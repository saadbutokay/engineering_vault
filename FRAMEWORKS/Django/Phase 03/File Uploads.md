## Overview

A file upload is the most dangerous kind of form input. The user controls the file name, the declared type, the size, and every byte of the content, and your server will store it and often hand it back to other people. Uploads have been the entry point for stored cross-site scripting, remote code execution, disk exhaustion, and data leaks. Django gives you the building blocks (`FileField`, `ImageField`, upload handlers, storage backends), but safe handling is a design decision you must make. This note covers `FileField` and `ImageField`, `request.FILES`, upload handlers and size limits, validating a file's type by its content rather than its name, generated file names and path traversal, storage backends and `django-storages` with S3, serving private files, and the security rules for user uploads. It ends with a tested book cover upload pipeline that verifies content, renames files, enforces limits, and cleans up the old file.

## Core concepts

### FileField and ImageField

`FileField` stores a path to a file in the database column and the file itself in a storage backend. `ImageField` is a `FileField` that also verifies the file is an image using Pillow (`uv add pillow`). The model attribute is a `FieldFile`, which gives you `.name`, `.url`, `.path` (for local storage), `.size`, `.open()`, `.save()`, and `.delete()`.

### upload_to

`upload_to` tells Django where under the storage root to put new files. It can be a folder string with date placeholders (`"covers/%Y/%m"`) or a function `(instance, filename) -> str` that builds the full relative path.

### request.FILES and multipart forms

Files arrive in `request.FILES`, separate from `request.POST`. The HTML form must use `method="post"` and `enctype="multipart/form-data"`, otherwise the browser sends only the file name. Pass both to the form: `MyForm(request.POST, request.FILES)`.

### UploadedFile

Each uploaded file is an `UploadedFile` with `name` and `size` and a `content_type` that the client chose. Small files are kept in memory (`InMemoryUploadedFile`), larger ones go to a temporary file on disk (`TemporaryUploadedFile`).

### Upload handlers and size limits

- `FILE_UPLOAD_MAX_MEMORY_SIZE` (default 2.5 MB): files larger than this stream to a temporary file instead of memory.
- `DATA_UPLOAD_MAX_MEMORY_SIZE` (default 2.5 MB): maximum size of the non-file part of a request body.
- `DATA_UPLOAD_MAX_NUMBER_FIELDS` and `DATA_UPLOAD_MAX_NUMBER_FILES`: limits on how many fields and files one request may contain.
- `FILE_UPLOAD_PERMISSIONS` and `FILE_UPLOAD_DIRECTORY_PERMISSIONS`: file modes on disk.

Django does not limit the total size of a file upload by itself. That limit belongs to the web server in front of it and to your validators.

### Validating by content, not by name

The file name, extension, and `content_type` are all supplied by the client and mean nothing. A `.png` file can contain a script. Trust only what you can verify from the bytes: open the file with a real parser (Pillow for images), check magic numbers for formats such as PDF, and reject anything that does not match your allowlist.

### Generated file names and path traversal

User-supplied file names can contain `../` sequences, absolute paths, unusual characters, or very long names. Django sanitizes names (`get_valid_filename`) and refuses paths that escape the storage root, but the safest approach is to ignore the client's name completely and generate your own, for example a random UUID plus an extension you chose from the verified content.

### Storage backends

Files are stored through a storage class configured in `STORAGES["default"]`: `FileSystemStorage` writes under `MEDIA_ROOT`, and `django-storages` provides backends for S3-compatible object storage and others. Application code uses the same `FieldFile` API with any backend.

### Serving private files

Public media can be served directly by the web server or a CDN. Files that only some users may see must be served through a view that checks permission, or through short-lived signed URLs (Phase 11). Never place private files under a publicly served directory.

### Security of uploaded content

Beyond type confusion, uploads can carry active content (SVG and HTML files can run scripts in the browser), huge images that exhaust memory when decoded, malware aimed at other users, and enormous bodies that fill the disk.

## Basic syntax examples

### A model with an uploaded file

```python
from django.db import models


class Document(models.Model):
    title = models.CharField(max_length=200)
    attachment = models.FileField(upload_to="documents/%Y/%m/")
    photo = models.ImageField(upload_to="photos/", blank=True)
```

### The HTML form

```html
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form }}
  <button type="submit">Upload</button>
</form>
```

### The view

```python
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render


def upload(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        form = DocumentForm(request.POST, request.FILES)
        if form.is_valid():
            form.save()
            return redirect("documents:list")
    else:
        form = DocumentForm()
    return render(request, "documents/upload.html", {"form": form})
```

### Using a stored file

```python
document = Document.objects.get(pk=1)
document.attachment.name       # 'documents/2026/03/report.pdf'
document.attachment.url        # '/media/documents/2026/03/report.pdf'
document.attachment.size       # bytes
with document.attachment.open("rb") as handle:
    data = handle.read()
```

### upload_to as a function

```python
import uuid
from pathlib import PurePosixPath


def document_path(instance: "Document", filename: str) -> str:
    suffix = PurePosixPath(filename).suffix.lower()
    return f"documents/{uuid.uuid4().hex}{suffix}"
```

### Upload settings

```python
# Memory and request limits
FILE_UPLOAD_MAX_MEMORY_SIZE = 2_621_440        # 2.5 MB, then stream to disk
DATA_UPLOAD_MAX_MEMORY_SIZE = 2_621_440        # non-file request body
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1_000
DATA_UPLOAD_MAX_NUMBER_FILES = 20

# File modes on disk
FILE_UPLOAD_PERMISSIONS = 0o640
FILE_UPLOAD_DIRECTORY_PERMISSIONS = 0o750
```

### The matching web server limit

```nginx
server {
    client_max_body_size 5m;   # reject larger bodies before Django sees them
}
```

### Private files served through a view

```python
from django.conf import settings
from django.core.files.storage import FileSystemStorage
from django.db import models
from django.http import FileResponse, Http404, HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404

private_storage = FileSystemStorage(location=settings.PRIVATE_MEDIA_ROOT)


class Invoice(models.Model):
    owner_email = models.EmailField()
    pdf = models.FileField(upload_to="invoices/", storage=private_storage)


def download_invoice(request: HttpRequest, pk: int) -> HttpResponse:
    invoice = get_object_or_404(Invoice, pk=pk)
    # ... check that request.user may see this invoice (Phase 4) ...
    if not invoice.pdf:
        raise Http404("No file")
    return FileResponse(
        invoice.pdf.open("rb"),
        as_attachment=True,
        filename=f"invoice-{invoice.pk}.pdf",
    )
```

### Handing the transfer to the web server

```python
def download_invoice_fast(request: HttpRequest, pk: int) -> HttpResponse:
    invoice = get_object_or_404(Invoice, pk=pk)
    # ... permission check ...
    response = HttpResponse()
    response["X-Accel-Redirect"] = f"/protected/{invoice.pdf.name}"
    response["Content-Type"] = "application/pdf"
    response["Content-Disposition"] = f'attachment; filename="invoice-{invoice.pk}.pdf"'
    return response
```

```nginx
location /protected/ {
    internal;                     # only reachable through X-Accel-Redirect
    alias /srv/app/private/;
}
```

### Object storage with django-storages

```bash
uv add "django-storages[s3]"
```

```python
STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3.S3Storage",
        "OPTIONS": {
            "bucket_name": "my-app-media",
            "location": "media",
            "file_overwrite": False,
            "querystring_auth": True,    # signed, expiring URLs
        },
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

## Deeper behavior

### How an upload is processed

The multipart parser splits the request body into fields and files and feeds each file through the upload handlers. The memory handler collects small files in memory. When a file exceeds `FILE_UPLOAD_MAX_MEMORY_SIZE`, the temporary file handler writes it to a temporary file on disk instead. By the time your view runs, the parsing is complete. Use `UploadedFile.chunks()` to read large files in pieces instead of `read()`, which loads everything into memory.

### Django does not limit the request size

`DATA_UPLOAD_MAX_MEMORY_SIZE` limits only the non-file part of the body. A client can send a very large file and Django will keep receiving it. Set `client_max_body_size` (Nginx) or the equivalent in your proxy, and add size validators in the form for a clear user-facing message. Doing both gives you protection before parsing and good errors after.

### Why content_type cannot be trusted

`upload.content_type` is the value of the `Content-Type` header the client attached to that part. Attackers set it freely. Likewise, browsers decide file extensions from the user's system. Verification must read the bytes.

### Verifying images with Pillow

`ImageField` (and `forms.ImageField`) opens the file with Pillow and calls `verify()`, which catches many corrupted or non-image files. That does not narrow the format: Pillow decodes GIF, BMP, TIFF, and other types. Check `image.format` against your allowlist, as the example below does. Decoding is also where decompression bombs bite: a small file that expands to billions of pixels. Pillow warns above `Image.MAX_IMAGE_PIXELS` and raises `DecompressionBombError` at twice that limit, and you should enforce your own tighter maximum before decoding more than the header. Reading `image.size` only parses the header, so check dimensions first.

### Re-encoding

For maximum safety, decode the validated image and write a fresh copy in a standard format. This strips embedded metadata (including location data) and any trailing payload hidden after the image data, at the cost of CPU time and some quality. Many sites do this anyway when generating thumbnails (Phase 11).

### Names, paths, and traversal

`FileSystemStorage` calls `get_valid_name()` on the file name and refuses paths that would leave `MEDIA_ROOT` by raising `SuspiciousFileOperation`. If the target already exists, `get_available_name()` appends a random suffix rather than overwriting. These protections are good defaults, but generating your own names avoids every class of problem: collisions, Unicode confusion, overlong names, and leaking personal information that users put in file names.

### Extensions

Keep an allowlist of extensions per upload kind and map the verified format to the extension yourself. A file called `shell.php` that contains a valid PNG should be stored as `.png`, never `.php`. Never allow upload folders to be executed by the web server.

### Files and database transactions

Writing a file and saving a database row are separate systems with no shared transaction. If the file is written and the transaction then rolls back, you have an orphan file. If you delete the old file and the transaction rolls back, you lost a file the row still references. The safe order is: write the new file, update the row, delete the old file only after the commit succeeds (`transaction.on_commit`), and delete the new file if the update fails.

### Django does not delete files for you

Deleting a model instance or replacing its file leaves the old file in storage. Clean up explicitly in services, or with a periodic job that finds unreferenced files. Do this deliberately rather than in signals, which also fire for rows removed by rolled-back transactions only if you register them carelessly.

### Active content and where you serve uploads from

Browsers execute scripts inside HTML and SVG files. If users can upload them and you serve them from your own domain, an uploaded file can read your site's cookies and act as the user. Prefer rejecting such types. If you must allow them, serve user content from a separate domain (not a subdomain sharing your cookies), send `Content-Disposition: attachment`, and add `X-Content-Type-Options: nosniff`, which Django's `SecurityMiddleware` sets by default through `SECURE_CONTENT_TYPE_NOSNIFF`.

### Serving private files

`FileResponse` streams a file from Django, which is simple and fine for modest traffic. At scale, let the web server send the bytes: Nginx's `X-Accel-Redirect` or Apache's `X-Sendfile` hand the transfer off after your permission check. For object storage, generate a short-lived signed URL and redirect to it (note 11.4).

### django-storages behavior

With S3 storage, `FieldFile.path` does not exist because there is no local path. Use `.url` and `.open()`. Settings such as `file_overwrite=False` keep it from replacing objects with the same key, and `querystring_auth=True` produces signed, expiring URLs for private buckets. The bucket itself must be private unless every file is public.

### Malware scanning

If users share files with other users, add a scanning step (for example ClamAV) before the file becomes visible: store it as pending, scan in a background task (Phase 9), then mark it available. Scanning reduces risk and does not replace type validation and safe serving.

## Industry-level snippet

A cover image pipeline for books. It rejects oversized files before parsing, verifies that the content is a real JPEG, PNG, or WebP image within pixel limits, ignores the client's file name in favor of a generated one with an extension derived from the content, stores the new file, and removes the old one only after the database commit. The view accepts the upload with a plain form. Everything is tested with real images generated in memory.

Add to `apps/bookstore/models.py`:

```python
from apps.bookstore.uploads import book_cover_path

# in Book
cover = models.ImageField(upload_to=book_cover_path, blank=True)
```

Install Pillow with `uv add pillow` and run `makemigrations`.

`apps/bookstore/uploads.py`:

```python
"""Safe handling of uploaded cover images."""

import logging
import uuid
from dataclasses import dataclass
from pathlib import PurePosixPath
from typing import TYPE_CHECKING

from django.core.exceptions import ValidationError
from django.core.files.uploadedfile import UploadedFile
from PIL import Image, UnidentifiedImageError

if TYPE_CHECKING:
    from apps.bookstore.models import Book

logger = logging.getLogger(__name__)

MAX_COVER_BYTES = 2 * 1024 * 1024
MAX_COVER_PIXELS = 4000 * 4000
ALLOWED_IMAGE_FORMATS: dict[str, str] = {
    "JPEG": ".jpg",
    "PNG": ".png",
    "WEBP": ".webp",
}
ALLOWED_SUFFIXES = frozenset(ALLOWED_IMAGE_FORMATS.values())


@dataclass(frozen=True)
class ImageInfo:
    format: str
    width: int
    height: int


def inspect_image(upload: UploadedFile) -> ImageInfo:
    """Verify an upload by its content and return what it really is.

    Raises:
        ValidationError: if the file is too large, is not an image, is not
            an allowed format, or has too many pixels.
    """
    if upload.size > MAX_COVER_BYTES:
        raise ValidationError(
            "Cover images must be at most %(max_kb)d KB.",
            code="file_too_large",
            params={"max_kb": MAX_COVER_BYTES // 1024},
        )

    try:
        upload.seek(0)
        with Image.open(upload) as image:
            detected = image.format
            width, height = image.size  # reads only the header
            if width * height > MAX_COVER_PIXELS:
                raise ValidationError(
                    "Cover images must not exceed %(max_pixels)d pixels.",
                    code="image_too_large",
                    params={"max_pixels": MAX_COVER_PIXELS},
                )
            image.verify()
    except (
        UnidentifiedImageError,
        OSError,
        SyntaxError,
        ValueError,
        Image.DecompressionBombError,
    ) as exc:
        raise ValidationError("Upload a valid image.", code="invalid_image") from exc
    finally:
        upload.seek(0)

    if detected is None or detected not in ALLOWED_IMAGE_FORMATS:
        raise ValidationError(
            "Only JPEG, PNG, and WebP images are accepted.", code="unsupported_format"
        )
    return ImageInfo(format=detected, width=width, height=height)


def validate_cover_upload(upload: UploadedFile) -> None:
    """Form validator wrapper around :func:`inspect_image`."""
    inspect_image(upload)


def safe_cover_filename(upload: UploadedFile) -> str:
    """Return a random file name whose extension comes from the verified content."""
    info = inspect_image(upload)
    return f"{uuid.uuid4().hex}{ALLOWED_IMAGE_FORMATS[info.format]}"


def book_cover_path(instance: "Book", filename: str) -> str:
    """``upload_to`` function that never uses the client's file name."""
    suffix = PurePosixPath(filename).suffix.lower()
    if suffix not in ALLOWED_SUFFIXES:
        suffix = ".img"
    return f"books/covers/{uuid.uuid4().hex}{suffix}"
```

`apps/bookstore/services/covers.py`:

```python
"""Replacing a book's cover image."""

import logging
from functools import partial

from django.core.files.uploadedfile import UploadedFile
from django.db import transaction

from apps.bookstore.models import Book
from apps.bookstore.uploads import safe_cover_filename

logger = logging.getLogger(__name__)


def set_cover(*, book: Book, upload: UploadedFile) -> str:
    """Store ``upload`` as the book's cover and return the new storage name.

    The old file is deleted only after the database change has committed.
    A new file that could not be recorded is removed again.

    Raises:
        ValidationError: if the upload is not an acceptable image.
    """
    name = safe_cover_filename(upload)  # verifies the content
    storage = book.cover.storage
    old_name = book.cover.name
    new_name = ""
    try:
        with transaction.atomic():
            book.cover.save(name, upload, save=False)
            new_name = book.cover.name
            book.save(update_fields=["cover"])
            if old_name:
                transaction.on_commit(partial(storage.delete, old_name))
    except Exception:
        if new_name:
            storage.delete(new_name)
        logger.exception("Could not save cover for book %s", book.pk)
        raise
    logger.info("Set cover %s for book %s", new_name, book.pk)
    return new_name
```

`apps/bookstore/forms.py` (addition):

```python
from apps.bookstore.uploads import validate_cover_upload


class CoverUploadForm(forms.Form):
    cover = forms.ImageField(
        validators=[validate_cover_upload],
        widget=forms.ClearableFileInput(attrs={"accept": "image/jpeg,image/png,image/webp"}),
    )
```

The `accept` attribute only helps the file picker. It is not validation.

`apps/bookstore/views.py` (addition):

```python
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_http_methods

from apps.bookstore.forms import CoverUploadForm
from apps.bookstore.services.covers import set_cover


@require_http_methods(["GET", "POST"])
def upload_cover(request: HttpRequest, pk: int) -> HttpResponse:
    book = get_object_or_404(Book.objects.alive(), pk=pk)
    form = CoverUploadForm(request.POST or None, request.FILES or None)
    if request.method == "POST" and form.is_valid():
        set_cover(book=book, upload=form.cleaned_data["cover"])
        return redirect("bookstore:upload_cover", pk=book.pk)
    return render(request, "bookstore/upload_cover.html", {"form": form, "book": book})
```

Add `path("<int:pk>/cover/", views.upload_cover, name="upload_cover")` to the app's `urlpatterns`, and the template `apps/bookstore/templates/bookstore/upload_cover.html`:

```html
<h1>Cover for {{ book.title }}</h1>
{% if book.cover %}<img src="{{ book.cover.url }}" alt="Current cover" width="120">{% endif %}
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form }}
  <button type="submit">Upload</button>
</form>
```

`apps/bookstore/tests/test_uploads.py`:

```python
import io
import re
import tempfile

from django.core.exceptions import ValidationError
from django.core.files.uploadedfile import SimpleUploadedFile
from django.test import SimpleTestCase, TestCase, override_settings
from django.urls import reverse
from PIL import Image

from apps.bookstore.forms import CoverUploadForm
from apps.bookstore.models import Book
from apps.bookstore.services.covers import set_cover
from apps.bookstore.uploads import (
    book_cover_path,
    safe_cover_filename,
    validate_cover_upload,
)

from .factories import make_book


def image_bytes(
    fmt: str = "PNG", size: tuple[int, int] = (8, 8), mode: str = "RGB"
) -> bytes:
    buffer = io.BytesIO()
    Image.new(mode, size, 0).save(buffer, format=fmt)
    return buffer.getvalue()


def upload(name: str, content: bytes) -> SimpleUploadedFile:
    return SimpleUploadedFile(name, content, content_type="image/png")


def error_code(file: SimpleUploadedFile) -> str | None:
    try:
        validate_cover_upload(file)
    except ValidationError as exc:
        return exc.code
    return None


class ValidationTests(SimpleTestCase):
    def test_a_real_png_is_accepted_and_the_pointer_is_reset(self) -> None:
        file = upload("cover.png", image_bytes())
        validate_cover_upload(file)
        self.assertEqual(file.tell(), 0)

    def test_text_pretending_to_be_an_image_is_rejected(self) -> None:
        self.assertEqual(error_code(upload("cover.png", b"<?php echo 1; ?>")), "invalid_image")

    def test_disallowed_image_formats_are_rejected(self) -> None:
        self.assertEqual(error_code(upload("cover.png", image_bytes("GIF"))), "unsupported_format")

    def test_oversized_files_are_rejected_before_parsing(self) -> None:
        self.assertEqual(error_code(upload("cover.png", b"0" * (3 * 1024 * 1024))), "file_too_large")

    def test_images_with_too_many_pixels_are_rejected(self) -> None:
        big = image_bytes("PNG", size=(4001, 4001), mode="L")
        self.assertEqual(error_code(upload("cover.png", big)), "image_too_large")


class NamingTests(SimpleTestCase):
    def test_extension_comes_from_the_content_not_the_client_name(self) -> None:
        php_named_png = safe_cover_filename(upload("shell.php", image_bytes("PNG")))
        self.assertTrue(php_named_png.endswith(".png"))
        self.assertNotIn("php", php_named_png)

        png_named_jpeg = safe_cover_filename(upload("photo.png", image_bytes("JPEG")))
        self.assertTrue(png_named_jpeg.endswith(".jpg"))

    def test_upload_path_ignores_traversal_and_unknown_suffixes(self) -> None:
        book = Book(title="Unsaved")
        path = book_cover_path(book, "../../etc/evil.png")
        self.assertRegex(path, r"^books/covers/[0-9a-f]{32}\.png$")
        self.assertTrue(book_cover_path(book, "x.php").endswith(".img"))


class StorageTestCase(TestCase):
    def setUp(self) -> None:
        super().setUp()
        directory = tempfile.TemporaryDirectory()
        self.addCleanup(directory.cleanup)
        override = override_settings(MEDIA_ROOT=directory.name)
        override.enable()
        self.addCleanup(override.disable)


class SetCoverTests(StorageTestCase):
    def test_cover_is_stored_under_a_generated_name(self) -> None:
        book = make_book()
        name = set_cover(book=book, upload=upload("../../My Cover.PNG", image_bytes()))
        self.assertRegex(name, r"^books/covers/[0-9a-f]{32}\.png$")
        self.assertTrue(book.cover.storage.exists(name))
        book.refresh_from_db()
        self.assertEqual(book.cover.name, name)

    def test_replacing_a_cover_deletes_the_old_file_after_commit(self) -> None:
        book = make_book()
        first = set_cover(book=book, upload=upload("a.png", image_bytes()))
        with self.captureOnCommitCallbacks(execute=True):
            second = set_cover(book=book, upload=upload("b.png", image_bytes("JPEG")))
        storage = book.cover.storage
        self.assertFalse(storage.exists(first))
        self.assertTrue(storage.exists(second))

    def test_invalid_upload_changes_nothing(self) -> None:
        book = make_book()
        with self.assertRaises(ValidationError):
            set_cover(book=book, upload=upload("a.png", b"not an image"))
        book.refresh_from_db()
        self.assertFalse(book.cover)


class CoverViewTests(StorageTestCase):
    def test_valid_upload_redirects_and_saves(self) -> None:
        book = make_book()
        url = reverse("bookstore:upload_cover", kwargs={"pk": book.pk})
        response = self.client.post(url, {"cover": upload("c.png", image_bytes())})
        self.assertRedirects(response, url)
        book.refresh_from_db()
        self.assertTrue(book.cover)

    def test_invalid_upload_re_renders_with_an_error(self) -> None:
        book = make_book()
        url = reverse("bookstore:upload_cover", kwargs={"pk": book.pk})
        response = self.client.post(url, {"cover": upload("c.png", b"nope")})
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Upload a valid image")

    def test_missing_file_is_a_required_error(self) -> None:
        book = make_book()
        response = self.client.post(reverse("bookstore:upload_cover", kwargs={"pk": book.pk}), {})
        self.assertContains(response, "This field is required")


class CoverFormTests(SimpleTestCase):
    def test_form_runs_the_content_validator(self) -> None:
        form = CoverUploadForm(files={"cover": upload("c.png", image_bytes("GIF"))})
        self.assertFalse(form.is_valid())
        self.assertIn("cover", form.errors)
```

## Common mistakes

- Forgetting `enctype="multipart/form-data"` on the form, so only the file name arrives.
- Forgetting to pass `request.FILES` to the form.
- Trusting the extension or `content_type` of an upload instead of inspecting the bytes.
- Allowing SVG or HTML uploads and serving them from the main domain.
- Using the client's file name for storage instead of generating one.
- Relying on Django for an overall request size limit, and not configuring the web server.
- Reading a large upload with `read()` instead of `chunks()`.
- Decoding an untrusted image before checking its dimensions.
- Deleting the old file inside the transaction, or never deleting it, rather than after the commit.
- Placing private files in a directory the web server publishes.
- Serving large files through Python for every download instead of delegating to the web server or storage.
- Testing uploads against the real `MEDIA_ROOT` instead of a temporary directory.
- Assuming `FieldFile.path` exists on every storage backend.

## Practice exercises

1. Build a `Document` upload form that accepts only PDFs. Verify the content by checking the first five bytes for `%PDF-`, reject everything else, and write tests with a real PDF header and a renamed text file.
2. Write a validator that rejects a file whose size is zero bytes, and another that rejects names longer than 100 characters, with tests for both. Then decide whether the name check is needed when you generate names yourself.
3. Add a decompression bomb test: create an image with a width and height whose product exceeds `MAX_COVER_PIXELS` and show it is rejected from the header alone, without decoding the pixels.
4. Implement thumbnail generation with Pillow that re-encodes a validated cover as a 200 by 300 JPEG, strips metadata, and stores it under a generated name. Test that the stored image has the expected size and contains no EXIF data.
5. Write a management command that finds files under `books/covers/` that no `Book.cover` references and reports them, with a `--delete` option guarded by a minimum age. Test it with temporary storage.
6. Write a test proving that `set_cover` removes the newly written file again when the database update fails, by making `Book.save` raise an exception (for example with `unittest.mock.patch.object`), and check that the old file is still present.
7. Write a view that serves a private file through `FileResponse` only to a user who passes a placeholder permission check, returns 404 for missing files, and sets `Content-Disposition: attachment`. Test the three outcomes.

## Summary

Uploads put the user in control of the file name, type, size, and content, so every one of those must be verified or replaced. Use `enctype="multipart/form-data"` and `request.FILES`, limit sizes both in the web server and in validators, verify content with a real parser against an allowlist of formats, and enforce dimension limits before decoding. Generate your own file names with extensions derived from the verified content, never from the client. Treat file and database writes as separate systems: write the new file, update the row, and delete the old file only after the commit. Serve public files from the web server or a CDN, serve private files through a permission check or signed URLs, never serve user-controlled HTML or SVG from your main domain, and test uploads with real generated files in a temporary storage directory.
