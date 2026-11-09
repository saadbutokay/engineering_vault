## Overview

Knowing how a form validates is only half the job. The other half is the flow around it: what happens when the user submits, how errors are shown, what the browser does when they press refresh, how long lists are split into pages, how search forms keep their state, and how a page can update one section without reloading. A handful of well-established patterns solve these problems, and using them consistently makes your forms feel correct and avoids duplicate submissions and lost input. This note covers the Post/Redirect/Get pattern, the generic editing views (`CreateView`, `UpdateView`, `DeleteView`, `FormView`), handling form errors, pagination with `Paginator`, search and filter forms with GET, HTMX for partial page updates, progressive enhancement, and how to tell when a form should really be an API. It ends with a book page that shows paginated reviews and accepts a new review with and without JavaScript, plus a paginated search page.

## Core concepts

### Post/Redirect/Get

After a successful POST, do not render a page. Redirect to a GET URL instead.

1. The browser sends a POST with the form data.
2. The server processes it and responds with a redirect (302 or 303).
3. The browser requests the new URL with GET and shows the result.

Without the redirect, pressing refresh makes the browser resubmit the form, creating a duplicate review or a second order. With it, refresh simply reloads the confirmation page. When validation fails, do not redirect: re-render the page with the bound form so the user sees their input and the errors.

### The standard flow

A form view has three outcomes:

- GET: show an empty (unbound) form.
- POST and valid: process the data, then redirect.
- POST and invalid: show the bound form with errors, with a normal 200 status.

### Generic editing views

- `CreateView`: show a `ModelForm`, save a new object, redirect.
- `UpdateView`: the same for an existing object.
- `DeleteView`: show a confirmation page, delete on POST.
- `FormView`: handle a non-model form.

They implement the flow above. You customize them with `form_class`, `template_name`, `success_url` or `get_success_url()`, `form_valid()`, and `form_invalid()`.

### Feedback messages

The `django.contrib.messages` framework stores a one-time message in the session or a cookie so that the page after a redirect can display it: `messages.success(request, "Saved.")`. Templates iterate over `messages`.

### Pagination

`Paginator(queryset, per_page)` splits a result into pages. `paginator.get_page(number)` returns a `Page` and is forgiving: a missing or invalid number gives page 1, and a number past the end gives the last page. `paginator.page(number)` is strict and raises `PageNotAnInteger` and `EmptyPage`. A `Page` provides `has_next()`, `has_previous()`, `next_page_number()`, `number`, and iteration over its objects. Paginating an unordered queryset gives inconsistent pages, so always order it.

### Search and filter forms with GET

Search forms use `method="get"`, so the URL carries the query. That makes results shareable and bookmarkable and makes the back button work. Bind the form to `request.GET`, and keep the search terms in pagination links (the `{% querystring %}` tag does this since Django 5.1).

### HTMX and partial page updates

HTMX is a small JavaScript library that lets HTML attributes issue requests and swap the response into the page, without writing JavaScript. `hx-post` sends the form, `hx-target` picks the element to update, and `hx-swap` picks how to insert the response. The server stays in control: it returns HTML fragments, and Django templates and forms do the work. HTMX sends the header `HX-Request: true`, which lets the view return only the fragment.

### Progressive enhancement

Build the page so it works with plain HTML forms and full page loads, then layer HTMX on top. If JavaScript is unavailable or fails, the form still posts, redirects, and renders. Treat the HTMX behavior as an improvement, not the foundation.

### When a form should be an API instead

Use an HTML form when the response is a page for a human. Build a JSON API (Phase 7) when the client is a mobile app, a single-page application, or another server, when the same operation must be used by several clients, or when you need machine-readable errors, versioning, and tokens. An API and a form can share the same validation and service layer, which is why note 3.5 keeps business rules out of forms.

## Basic syntax examples

### Post/Redirect/Get in a function view

```python
from django.contrib import messages
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render

from apps.pages.forms import ContactForm


def contact(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        form = ContactForm(request.POST)
        if form.is_valid():
            # ... process form.cleaned_data ...
            messages.success(request, "Thanks, we will be in touch.")
            return redirect("pages:contact")      # Redirect: the next request is a GET
    else:
        form = ContactForm()                       # Get: empty form
    return render(request, "pages/contact.html", {"form": form})   # invalid: re-render
```

### Showing messages

```html
{% if messages %}
  <ul class="messages">
    {% for message in messages %}
      <li class="{{ message.tags }}">{{ message }}</li>
    {% endfor %}
  </ul>
{% endif %}
```

### CreateView and UpdateView

```python
from django.contrib import messages
from django.http import HttpResponse
from django.urls import reverse_lazy
from django.views.generic import CreateView, UpdateView

from apps.bookstore.forms import BookForm
from apps.bookstore.models import Book


class BookCreateView(CreateView[Book, BookForm]):
    form_class = BookForm
    template_name = "bookstore/book_form.html"
    success_url = reverse_lazy("bookstore:search")

    def form_valid(self, form: BookForm) -> HttpResponse:
        response = super().form_valid(form)
        messages.success(self.request, f"Added {form.instance.title}.")
        return response


class BookUpdateView(UpdateView[Book, BookForm]):
    form_class = BookForm
    template_name = "bookstore/book_form.html"
    queryset = Book.objects.alive()
    success_url = reverse_lazy("bookstore:search")
```

### DeleteView, adapted to a soft delete

```python
from django import forms
from django.http import HttpResponse, HttpResponseRedirect
from django.views.generic import DeleteView


class BookDeleteView(DeleteView[Book, forms.Form]):
    queryset = Book.objects.alive()
    template_name = "bookstore/book_confirm_delete.html"
    success_url = reverse_lazy("bookstore:search")

    def form_valid(self, form: forms.Form) -> HttpResponse:
        book = self.get_object()
        Book.objects.filter(pk=book.pk).soft_delete()   # keep the row, mark it deleted
        return HttpResponseRedirect(self.get_success_url())
```

Type parameters for the generic edit views depend on the version of `django-stubs`. If `mypy` rejects one of them, check the stub's class definition.

### FormView for a non-model form

```python
from django.views.generic import FormView

from apps.pages.forms import ContactForm


class ContactView(FormView[ContactForm]):
    form_class = ContactForm
    template_name = "pages/contact.html"
    success_url = reverse_lazy("pages:contact")

    def form_valid(self, form: ContactForm) -> HttpResponse:
        send_contact_message(form.cleaned_data)
        return super().form_valid(form)
```

### Paginating a queryset

```python
from django.core.paginator import Paginator

paginator = Paginator(Book.objects.order_by("title"), 10)
page = paginator.get_page(request.GET.get("page"))   # never raises

page.number            # current page number
page.has_next()
page.paginator.num_pages
for book in page:      # the objects on this page
    ...
```

### Pagination controls that keep the search terms

```html
{% if page_obj.has_other_pages %}
  <nav aria-label="Pages">
    {% if page_obj.has_previous %}
      <a href="{% querystring page=page_obj.previous_page_number %}">Previous</a>
    {% endif %}
    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
    {% if page_obj.has_next %}
      <a href="{% querystring page=page_obj.next_page_number %}">Next</a>
    {% endif %}
  </nav>
{% endif %}
```

### An HTMX-enhanced form

```html
<div hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
  <section id="reviews">
    <form method="post"
          action="{% url 'bookstore:book_detail' pk=book.pk %}"
          hx-post="{% url 'bookstore:book_detail' pk=book.pk %}"
          hx-target="#reviews"
          hx-swap="outerHTML">
      {% csrf_token %}
      {{ form }}
      <button type="submit">Post review</button>
    </form>
  </section>
</div>
```

Without JavaScript this is an ordinary form that posts to `action`. With HTMX it posts in the background and replaces the `#reviews` section with the response.

### Detecting an HTMX request on the server

```python
def is_htmx(request: HttpRequest) -> bool:
    return request.headers.get("HX-Request") == "true"
```

### Useful HTMX response headers

```python
response = HttpResponse(status=204)
response["HX-Redirect"] = "/books/search/"          # full redirect from an HTMX request
response["HX-Trigger"] = "review-saved"             # fire a client-side event
```

## Deeper behavior

### Why the redirect matters

A browser remembers the last navigation. If that was a POST that returned a page, refresh and back/forward ask to resend the form, and a careless user confirms. The redirect replaces the history entry's request with a GET. Use 302 (the default) or 303 See Other; 303 states explicitly that the follow-up must be a GET.

### Redirect targets

Send the user to the thing they just created or edited, to the list it belongs to, or back to where they came from. If you redirect to a URL provided by the request (a `next` parameter), validate it with `url_has_allowed_host_and_scheme()`, otherwise attackers can use your site for open redirects (note 6.3).

### Invalid forms and status codes

Browsers and HTMX treat 2xx as success. Django re-renders invalid forms with status 200 by default, which keeps the browser and HTMX happy. Some teams use 400 or 422 for invalid submissions, which is more precise but requires configuring HTMX to swap error responses (by default it ignores 4xx and 5xx responses).

### Duplicate submissions and races

PRG protects against refresh, not against double clicks or two tabs. A unique constraint (note 2.10) is the real protection, and the view must handle its `IntegrityError` by turning it into a form error, as the example does. For operations that must happen once, use idempotency keys (note 2.9) and disable the submit button on the first click as a convenience.

### Messages framework details

Messages are stored by a storage backend (cookie, session, or a fallback combining them) and are removed from storage once they have been iterated. If a template never loops over `messages`, they stay queued and appear on a later page. HTMX fragment responses that do not render messages can therefore leave stale messages behind, so for fragment responses show feedback inline instead.

### The generic views in detail

`CreateView` and `UpdateView` call `get_form_class()` and `get_form_kwargs()` (which supplies `instance`, `data`, and `files`), then `form_valid()` saves and redirects, while `form_invalid()` re-renders. Override `get_form_kwargs()` to pass extra arguments such as the current user, and `get_queryset()` to restrict which objects can be edited (note 4.3). `DeleteView` in Django 4.0 and later is form-based, so `form_valid()` performs the delete, and overriding it is how you turn a delete into a soft delete.

### Choosing between a function and a class

The generic views save code when your flow matches the standard one. When the view must coordinate several forms, handle HTMX variants, or call services with special error handling, a function-based view with an explicit flow is usually clearer, as in the example below.

### Pagination in detail

- `Paginator` runs one `COUNT(*)` query and one query for the page. On very large tables `COUNT` can be slow, and for infinite scroll or APIs, cursor-based pagination (note 7.6) is preferable.
- Paginating an unordered queryset gives a warning, because the database may return rows in a different order on each query, causing duplicates and gaps across pages. Always call `order_by` with a unique tie-breaker.
- Page numbers in URLs are user input. `get_page()` makes them safe by returning page 1 or the last page. Use `page()` only when you want an invalid page to be a 404.
- Offset pagination gets slower for deep pages because the database still skips all earlier rows. Most interfaces never reach those pages, and bots that crawl them can hurt, so consider limiting the maximum page.

### GET forms and state

Because the query string holds the form state, the page is stateless on the server: any request with the same URL gives the same result. Do not put sensitive data in GET parameters, which end up in logs, history, and referrer headers.

### HTMX in practice

HTMX requests are ordinary HTTP requests, so Django middleware, CSRF protection, authentication, and forms all work unchanged. CSRF needs the token on every unsafe request: set it once on a wrapper element with `hx-headers` (or on `<body>`), as the example does. The server decides what to return: the full page for normal requests and a fragment for HTMX requests, usually by rendering a different template. With Django 6.0, template partials let one file define both (note 1.4). On Django 5.2 LTS, put the fragment in its own template and `{% include %}` it from the full page.

### Progressive enhancement checks

Test the form in both modes: the plain POST with redirect, and the HTMX request with the fragment response. If both behave, the page degrades gracefully. Also make sure fragments contain everything the user needs (errors, success message, fresh form), because they replace part of the page.

### Server-driven versus API-driven

HTMX keeps the logic on the server and returns HTML, which is simple to build and secure by default because Django's escaping and CSRF still apply. A single-page application moves rendering to the browser and talks to a JSON API, which is the right choice when you need rich client-side state or multiple clients, at the cost of a second codebase, API design, and token authentication. Choose consciously, and keep business rules in services so either approach can reuse them.

## Industry-level snippet

A book page with paginated reviews and a review form. The same view serves three cases: a normal GET, a normal POST with Post/Redirect/Get, and an HTMX POST that returns only the review section. It handles an invalid form, a duplicate caught by the form, and a duplicate caught by the database constraint. The search page from note 3.1 gains pagination that keeps the filters in its links.

`apps/bookstore/views.py`:

```python
import logging
from typing import Any

from django.contrib import messages
from django.core.paginator import Paginator
from django.db import IntegrityError, transaction
from django.db.models import QuerySet
from django.http import HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_http_methods

from apps.bookstore.forms import BookSearchForm, ReviewForm
from apps.bookstore.models import Book
from apps.bookstore.selectors import BookFilters, search_books

logger = logging.getLogger(__name__)

REVIEWS_PER_PAGE = 5
SEARCH_PAGE_SIZE = 10


def is_htmx(request: HttpRequest) -> bool:
    return request.headers.get("HX-Request") == "true"


def review_context(
    book: Book,
    form: ReviewForm,
    page_number: str | None,
    *,
    just_reviewed: bool = False,
) -> dict[str, Any]:
    reviews = book.reviews.order_by("-created_at", "-id")
    page_obj = Paginator(reviews, REVIEWS_PER_PAGE).get_page(page_number)
    return {
        "book": book,
        "form": form,
        "page_obj": page_obj,
        "just_reviewed": just_reviewed,
    }


@require_http_methods(["GET", "POST"])
def book_detail(request: HttpRequest, pk: int) -> HttpResponse:
    book = get_object_or_404(Book.objects.alive().select_related("author"), pk=pk)
    htmx = is_htmx(request)

    if request.method == "POST":
        form = ReviewForm(request.POST, book=book)
        if form.is_valid():
            try:
                with transaction.atomic():
                    form.save()
            except IntegrityError:
                # A concurrent identical submission won the race for the
                # unique constraint, so report it like any other duplicate.
                logger.info("Duplicate review rejected by the database for book %s", pk)
                form.add_error("reviewer_name", "You have already reviewed this book.")
            else:
                if htmx:
                    context = review_context(
                        book, ReviewForm(book=book), None, just_reviewed=True
                    )
                    return render(request, "bookstore/_review_section.html", context)
                messages.success(request, "Thanks for your review.")
                return redirect("bookstore:book_detail", pk=book.pk)
    else:
        form = ReviewForm(book=book)

    context = review_context(book, form, request.GET.get("page"))
    template = (
        "bookstore/_review_section.html" if htmx else "bookstore/book_detail.html"
    )
    return render(request, template, context)


def book_search(request: HttpRequest) -> HttpResponse:
    form = BookSearchForm(request.GET or None)
    books: QuerySet[Book]
    if form.is_bound and form.is_valid():
        books = search_books(form.to_filters())
    elif form.is_bound:
        books = Book.objects.none()
    else:
        books = search_books(BookFilters())
    page_obj = Paginator(books, SEARCH_PAGE_SIZE).get_page(request.GET.get("page"))
    return render(request, "bookstore/book_search.html", {"form": form, "page_obj": page_obj})
```

`apps/bookstore/urls.py`:

```python
from django.urls import path

from . import views

app_name = "bookstore"

urlpatterns = [
    path("search/", views.book_search, name="search"),
    path("<int:pk>/", views.book_detail, name="book_detail"),
    path("<int:pk>/cover/", views.upload_cover, name="upload_cover"),
]
```

`apps/bookstore/templates/bookstore/book_detail.html`:

```html
<h1>{{ book.title }}</h1>
<p>by {{ book.author.name }}</p>
{% if messages %}
  <ul class="messages">
    {% for message in messages %}<li>{{ message }}</li>{% endfor %}
  </ul>
{% endif %}
<div hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
  {% include "bookstore/_review_section.html" %}
</div>
```

`apps/bookstore/templates/bookstore/_review_section.html`:

```html
<section id="reviews">
  <h2>Reviews</h2>
  {% if just_reviewed %}<p class="notice">Thanks for your review.</p>{% endif %}
  <ul class="review-list">
    {% for review in page_obj %}
      <li>{{ review.reviewer_name }}: {{ review.rating }}/5 {{ review.body }}</li>
    {% empty %}
      <li>No reviews yet.</li>
    {% endfor %}
  </ul>
  {% if page_obj.has_other_pages %}
    <nav aria-label="Review pages">
      {% if page_obj.has_previous %}
        <a href="{% querystring page=page_obj.previous_page_number %}">Newer</a>
      {% endif %}
      <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
      {% if page_obj.has_next %}
        <a href="{% querystring page=page_obj.next_page_number %}">Older</a>
      {% endif %}
    </nav>
  {% endif %}
  <form method="post"
        action="{% url 'bookstore:book_detail' pk=book.pk %}"
        hx-post="{% url 'bookstore:book_detail' pk=book.pk %}"
        hx-target="#reviews"
        hx-swap="outerHTML">
    {% csrf_token %}
    {{ form }}
    <button type="submit">Post review</button>
  </form>
</section>
```

`apps/bookstore/templates/bookstore/book_search.html`:

```html
<h1>Find a book</h1>
<form method="get" action="{% url 'bookstore:search' %}">
  {{ form }}
  <button type="submit">Search</button>
</form>
<ul>
  {% for book in page_obj %}
    <li>{{ book.title }} by {{ book.author.name }} ({{ book.availability }})</li>
  {% empty %}
    <li>No books found.</li>
  {% endfor %}
</ul>
{% if page_obj.has_other_pages %}
  <nav aria-label="Result pages">
    {% if page_obj.has_previous %}
      <a href="{% querystring page=page_obj.previous_page_number %}">Previous</a>
    {% endif %}
    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
    {% if page_obj.has_next %}
      <a href="{% querystring page=page_obj.next_page_number %}">Next</a>
    {% endif %}
  </nav>
{% endif %}
```

`apps/bookstore/tests/test_form_handling.py`:

```python
from unittest import mock

from django.contrib.messages import get_messages
from django.db import IntegrityError
from django.test import Client, TestCase
from django.urls import reverse

from apps.bookstore.forms import ReviewForm
from apps.bookstore.models import Review

from .factories import make_book

HTMX = {"HX-Request": "true"}


class BookDetailTests(TestCase):
    def setUp(self) -> None:
        self.book = make_book(title="Dune")
        self.url = reverse("bookstore:book_detail", kwargs={"pk": self.book.pk})

    def review(self, name: str, rating: int = 5) -> Review:
        return Review.objects.create(book=self.book, reviewer_name=name, rating=rating)

    def test_get_shows_the_book_reviews_and_an_empty_form(self) -> None:
        self.review("Ada")
        response = self.client.get(self.url)
        self.assertContains(response, "Dune")
        self.assertContains(response, "Ada")
        self.assertContains(response, "Post review")

    def test_reviews_are_paginated_newest_first(self) -> None:
        for index in range(12):
            self.review(f"Reviewer-{index:02d}")
        first = self.client.get(self.url)
        last = self.client.get(self.url, {"page": 3})
        self.assertContains(first, "Reviewer-11")
        self.assertNotContains(first, "Reviewer-06")
        self.assertContains(last, "Reviewer-01")
        self.assertContains(last, "Reviewer-00")
        self.assertNotContains(last, "Reviewer-11")

    def test_bad_page_numbers_do_not_crash(self) -> None:
        self.review("Ada")
        self.assertEqual(self.client.get(self.url, {"page": "abc"}).status_code, 200)
        self.assertEqual(self.client.get(self.url, {"page": 99}).status_code, 200)

    def test_valid_post_redirects_and_saves(self) -> None:
        response = self.client.post(
            self.url, {"reviewer_name": "Ada", "rating": "5", "body": "Great"}
        )
        self.assertRedirects(response, self.url)
        self.assertEqual(Review.objects.get().reviewer_name, "Ada")
        texts = [str(message) for message in get_messages(response.wsgi_request)]
        self.assertEqual(texts, ["Thanks for your review."])

    def test_invalid_post_re_renders_with_errors_and_saves_nothing(self) -> None:
        response = self.client.post(self.url, {"reviewer_name": "Ada", "rating": "9"})
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Select a valid choice")
        self.assertEqual(Review.objects.count(), 0)

    def test_duplicate_review_is_reported_by_the_form(self) -> None:
        self.review("Ada")
        response = self.client.post(self.url, {"reviewer_name": "ada", "rating": "4"})
        self.assertContains(response, "You have already reviewed this book.")
        self.assertEqual(Review.objects.count(), 1)

    def test_duplicate_caught_by_the_database_becomes_a_form_error(self) -> None:
        with mock.patch.object(ReviewForm, "save", side_effect=IntegrityError):
            response = self.client.post(
                self.url, {"reviewer_name": "Ada", "rating": "5"}
            )
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "You have already reviewed this book.")

    def test_htmx_post_returns_only_the_review_section(self) -> None:
        response = self.client.post(
            self.url, {"reviewer_name": "Ada", "rating": "5"}, headers=HTMX
        )
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Thanks for your review.")
        self.assertContains(response, 'id="reviews"')
        self.assertNotContains(response, "<h1>")
        self.assertEqual(Review.objects.count(), 1)

    def test_htmx_get_returns_the_fragment(self) -> None:
        response = self.client.get(self.url, headers=HTMX)
        self.assertContains(response, 'id="reviews"')
        self.assertNotContains(response, "<h1>")

    def test_csrf_token_is_required_for_posts(self) -> None:
        strict = Client(enforce_csrf_checks=True)
        response = strict.post(self.url, {"reviewer_name": "Ada", "rating": "5"})
        self.assertEqual(response.status_code, 403)

    def test_unknown_books_are_404(self) -> None:
        response = self.client.get(reverse("bookstore:book_detail", kwargs={"pk": 999_999}))
        self.assertEqual(response.status_code, 404)


class SearchPaginationTests(TestCase):
    def test_pages_split_results_and_keep_the_filters_in_links(self) -> None:
        for index in range(25):
            make_book(title=f"Match {index:02d}")
        make_book(title="Other")
        url = reverse("bookstore:search")

        first = self.client.get(url, {"q": "Match"})
        last = self.client.get(url, {"q": "Match", "page": 3})

        self.assertContains(first, "<li>", count=10)
        self.assertContains(first, "q=Match")      # the Next link keeps the search
        self.assertContains(last, "<li>", count=5)
        self.assertNotContains(last, "Other")

    def test_invalid_filters_show_errors_and_no_results(self) -> None:
        make_book(title="Dune")
        response = self.client.get(
            reverse("bookstore:search"), {"min_price": "20", "max_price": "5"}
        )
        self.assertContains(response, "Maximum price must not be below")
        self.assertNotContains(response, "Dune")
```

The view keeps one flow for three situations, and each branch has a test: refresh safety (redirect), error display (re-render), races (database constraint turned into a form error), and graceful enhancement (fragment responses).

## Common mistakes

- Rendering a page directly after a successful POST, so a refresh resubmits the form.
- Redirecting after an invalid POST, which throws away the user's input and the errors.
- Using GET forms for operations that change data, or POST forms for pure searches.
- Paginating an unordered queryset, or ordering by a non-unique column without a tie-breaker.
- Building pagination links that drop the search terms.
- Trusting the `page` parameter, instead of using `get_page()` or handling `EmptyPage`.
- Relying on a Python-level duplicate check alone, and not handling the `IntegrityError` from the constraint.
- Using HTMX without the CSRF header, then seeing 403 responses on every POST.
- Returning a full page to an HTMX request that expected a fragment, or the reverse.
- Making the page depend on JavaScript, so the form breaks when scripts fail to load.
- Leaving stale messages in storage after fragment responses that never display them.
- Redirecting to a user-supplied `next` URL without validating it.
- Building a form flow when the real consumer is another program that needs a JSON API.

## Practice exercises

1. Remove the redirect from a form view, submit the form, press refresh, and observe the duplicate. Restore the redirect and compare.
2. Write a `FormView` for a "contact the store" form that uses the Post/Redirect/Get pattern and a success message, and test the three outcomes (GET, valid POST, invalid POST).
3. Add pagination to a list of orders with 20 per page, ordered by creation time and id. Write a test that proves a row is never repeated or skipped across pages when two orders share a timestamp.
4. Add a `next` parameter to the review form's redirect, validate it with `url_has_allowed_host_and_scheme`, and test both a safe local path and an external URL.
5. Write a test that posts the review form twice in quick succession using two threads against PostgreSQL, and shows that exactly one review is created and the other request renders the duplicate error.
6. Turn the search page into an HTMX-enhanced page: typing in the search box updates the results list with a fragment response, while the page still works without JavaScript. Test both the full page and the fragment.
7. List three features of a hypothetical "order history" screen and decide for each whether it should be an HTML form flow or a JSON API, explaining your reasoning in terms of clients, error formats, and reuse of the service layer.

## Summary

After a successful POST, redirect to a GET URL (Post/Redirect/Get) so refresh cannot resubmit, and re-render the bound form with a normal response when validation fails. The generic edit views implement this flow and are customized through `form_valid`, `form_invalid`, and `get_success_url`, but explicit function views are clearer when the flow has variants. Use `Paginator.get_page` on ordered querysets for safe pagination, GET forms for searches so state lives in the URL, and `{% querystring %}` to keep filters across pages. Handle database constraint errors as form errors to cover races. HTMX adds partial updates on top of ordinary forms: send the CSRF header, return fragments for `HX-Request`, and keep the page working without JavaScript. Choose a JSON API instead when the client is not a person looking at a page, and keep business rules in services so both can share them.
