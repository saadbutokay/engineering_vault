## Overview

Every Django application is, at its core, a program that receives an HTTP request and returns an HTTP response. Before learning any framework, you need a clear mental model of what travels over the wire, what each part means, and which responsibilities belong to the browser, the network, and your code. This note builds that model using small Python programs and `curl`, so that later, when Django hides the details, you know what it is hiding.

You already know Python basics. Nothing here requires Django; everything uses the standard library.

---
## Core concepts

### Clients and servers

- A client starts a conversation. Browsers, mobile apps, `curl`, and other servers can all be clients.
- A server waits for requests, does work, and sends back a response.
- One program can be both: your Django app is a server to the browser and a client to a payment provider.
- HTTP is stateless: each request stands alone. State (logins, carts) is rebuilt from cookies, tokens, or database lookups.

### Anatomy of a URL

For `https://shop.example.com:8443/products/42?color=red&size=m#reviews`:

- Scheme: `https`, which protocol to use.
- Host: `shop.example.com`, resolved to an IP address by DNS.
- Port: `8443`, defaults to 80 for HTTP and 443 for HTTPS.
- Path: `/products/42`, which resource the server should find.
- Query string: `color=red&size=m`, extra key-value input.
- Fragment: `reviews`, used only by the browser and never sent to the server.

### Request and response structure

A request has a start line (method, target, version), headers, a blank line, and an optional body. A response has a status line (version, status code, reason phrase), headers, a blank line, and an optional body.

### Methods

- GET: read a resource. Must not change server state.
- POST: submit data or trigger an action. The general-purpose method for creating things.
- PUT: replace a resource entirely.
- PATCH: partially update a resource.
- DELETE: remove a resource.

### Safety and idempotency

- A safe method does not change server state (GET, HEAD, OPTIONS).
- An idempotent method has the same effect whether called once or many times (GET, HEAD, PUT, DELETE).
- POST is neither safe nor idempotent. Sending the same POST twice can create two orders.
- PATCH is not guaranteed to be idempotent.

### Status code families

| Family | Meaning       | Examples                            |
| ------ | ------------- | ----------------------------------- |
| 1xx    | Informational | 101 Switching Protocols             |
| 2xx    | Success       | 200 OK, 201 Created, 204 No Content |
| 3xx    | Redirection   | 301, 302, 304 Not Modified          |
| 4xx    | Client error  | 400, 401, 403, 404, 429             |
| 5xx    | Server error  | 500, 502, 503, 504                  |

### Headers, cookies, and bodies

- Headers are metadata: `Content-Type`, `Content-Length`, `Accept`, `Authorization`, `Cache-Control`, `Location`.
- A cookie is set by the server with `Set-Cookie` and returned by the browser in a `Cookie` header on later requests to the same site.
- Common body formats: `application/x-www-form-urlencoded` (simple HTML forms), `multipart/form-data` (file uploads), and `application/json` (APIs).

### HTTPS and TLS (conceptual)

TLS wraps HTTP in an encrypted channel. During the handshake, the server proves its identity with a certificate signed by a trusted authority, and both sides agree on session keys. HTTPS gives confidentiality, integrity, and server authentication. It does not make your application secure; it only protects data in transit.

### Why a web framework exists

Without a framework you would parse raw requests, route paths, decode forms, manage sessions, escape output, talk to a database, and handle errors, all by hand. A framework packages these solved problems so you write only application logic.

---
## Basic syntax examples

### A raw request and response
```text
GET /products/42?color=red HTTP/1.1
Host: shop.example.com
Accept: text/html
User-Agent: curl/8.5.0

```

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 27

<h1>Product 42 (red)</h1>
```

### Exploring with curl
```bash
# Show response headers and body, with request details
curl -v https://example.com/

# Headers only
curl -I https://example.com/

# POST form data
curl -X POST -d "name=Ada&role=admin" https://httpbin.org/post

# POST JSON
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada"}' \
  https://httpbin.org/post

# Follow redirects
curl -L http://example.com/old-path
```

On Windows PowerShell, use `curl.exe` explicitly, because `curl` is an alias for `Invoke-WebRequest`.

```powershell
curl.exe -I https://example.com/
```

### Taking a URL apart in Python
```python
from urllib.parse import parse_qs, urlsplit

url = "https://shop.example.com:8443/products/42?color=red&size=m#reviews"
parts = urlsplit(url)

print(parts.scheme)    # https
print(parts.hostname)  # shop.example.com
print(parts.port)      # 8443
print(parts.path)      # /products/42
print(parts.fragment)  # reviews

query: dict[str, list[str]] = parse_qs(parts.query)
print(query)  # {'color': ['red'], 'size': ['m']}
```

### Encoding request bodies
```python
import json
from urllib.parse import urlencode

form_body: str = urlencode({"name": "Ada Lovelace", "role": "admin"})
print(form_body)  # name=Ada+Lovelace&role=admin

json_body: str = json.dumps({"name": "Ada Lovelace", "roles": ["admin"]})
print(json_body)
```

### The smallest possible web server
```python
from http.server import BaseHTTPRequestHandler, HTTPServer


class HelloHandler(BaseHTTPRequestHandler):
    def do_GET(self) -> None:
        body = b"Hello, web"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


def main() -> None:
    server = HTTPServer(("127.0.0.1", 8000), HelloHandler)
    print("Listening on http://127.0.0.1:8000")
    server.serve_forever()


if __name__ == "__main__":
    main()
```

Run it, then in another terminal run `curl -v http://127.0.0.1:8000/`. Every line you see in the verbose output is something your handler produced or something curl added.

---
## Deeper behavior

### Redirects
A redirect is just a response with a 3xx status and a `Location` header.
```text
HTTP/1.1 302 Found
Location: /login?next=/dashboard
Content-Length: 0
```

- 301 and 308 are permanent; 302 and 307 are temporary.
- 301 and 302 allow clients to change POST to GET on the follow-up request; 307 and 308 forbid the change.
- 303 See Other explicitly means "fetch the result with GET". It is the correct response after a successful form POST (the Post/Redirect/Get pattern, covered in Phase 3).

### 401 versus 403, 400 versus 422
- 401 means "I do not know who you are" (unauthenticated).
- 403 means "I know who you are and you may not do this" (unauthorized).
- 400 means the request is malformed. 422 is often used when the request is well-formed but fails validation. APIs differ on this; choose one convention and keep it.

### Cookies in detail
```text
Set-Cookie: sessionid=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=1209600
```

- `HttpOnly`: JavaScript cannot read the cookie, limiting damage from XSS.
- `Secure`: sent only over HTTPS.
- `SameSite`: controls whether the cookie is sent on cross-site requests, which is a defense against CSRF.
- `Max-Age` or `Expires`: lifetime. Without either, it is a session cookie that disappears when the browser closes.

### Connection behavior
- HTTP/1.1 keeps connections open by default, so multiple requests reuse one TCP connection.
- HTTP/2 multiplexes many requests over one connection and compresses headers.
- Your Django code does not change between versions; the web server in front of it handles the protocol.

### Caching and conditional requests
A server can send `ETag` or `Last-Modified`. The client later asks "has it changed?" with `If-None-Match` or `If-Modified-Since`, and the server may answer `304 Not Modified` with no body, saving bandwidth.

### Developer tools
In a browser, open the Network tab, reload the page, and click any request. You can inspect request headers, response headers, timing, cookies, and the raw body. Learning to read this tab is the fastest way to debug anything web-related.

---
## Industry-level snippet

Backend code constantly calls other HTTP services. A production-grade call needs a timeout, input validation, explicit errors, logging, and tests. This standard-library client demonstrates the pattern; later phases replace it with `httpx`.

File `http_client.py`:
```python
"""Small, strictly validated JSON-over-HTTP GET client."""

import json
import logging
import urllib.error
import urllib.request
from dataclasses import dataclass
from typing import Any
from urllib.parse import urlsplit

logger = logging.getLogger(__name__)


class HttpClientError(Exception):
    """Base class for all errors raised by this module."""


class InvalidUrlError(HttpClientError):
    """The URL is not an absolute http or https URL."""


class TransportError(HttpClientError):
    """The request failed before a usable response was received."""


class UpstreamStatusError(HttpClientError):
    """The remote server answered with a 4xx or 5xx status."""

    def __init__(self, status: int, url: str) -> None:
        super().__init__(f"{url} returned status {status}")
        self.status = status
        self.url = url


@dataclass(frozen=True)
class ApiResult:
    status: int
    data: Any
    headers: dict[str, str]


def get_json(url: str, *, timeout: float = 5.0) -> ApiResult:
    """Fetch ``url`` and decode the body as JSON.

    Raises:
        InvalidUrlError: if the URL is not absolute http or https.
        ValueError: if ``timeout`` is not positive.
        UpstreamStatusError: if the server returns an error status.
        TransportError: for network failures and invalid JSON.
    """
    parts = urlsplit(url)
    if parts.scheme not in {"http", "https"} or not parts.netloc:
        raise InvalidUrlError(f"Unsupported URL: {url!r}")
    if timeout <= 0:
        raise ValueError("timeout must be positive")

    request = urllib.request.Request(  # noqa: S310 - scheme validated above
        url,
        headers={"Accept": "application/json"},
        method="GET",
    )
    try:
        with urllib.request.urlopen(request, timeout=timeout) as response:  # noqa: S310
            raw: bytes = response.read()
            status: int = response.status
            headers = {key.lower(): value for key, value in response.headers.items()}
    except urllib.error.HTTPError as exc:  # must come before URLError
        logger.warning("GET %s failed with status %s", url, exc.code)
        raise UpstreamStatusError(exc.code, url) from exc
    except (urllib.error.URLError, TimeoutError) as exc:
        logger.warning("GET %s failed: %s", url, exc)
        raise TransportError(f"Could not reach {url}") from exc

    try:
        data = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise TransportError(f"{url} did not return valid JSON") from exc

    logger.debug("GET %s -> %s (%d bytes)", url, status, len(raw))
    return ApiResult(status=status, data=data, headers=headers)
```

File `test_http_client.py`, using a real local server so no mocking is needed:
```python
import threading
from collections.abc import Iterator
from http.server import BaseHTTPRequestHandler, HTTPServer
from typing import Any

import pytest

from http_client import (
    InvalidUrlError,
    UpstreamStatusError,
    get_json,
)


class _Handler(BaseHTTPRequestHandler):
    def do_GET(self) -> None:
        if self.path == "/missing":
            self.send_response(404)
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        body = b'{"ok": true}'
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format: str, *args: Any) -> None:  # noqa: A002
        return  # keep test output quiet


@pytest.fixture
def base_url() -> Iterator[str]:
    server = HTTPServer(("127.0.0.1", 0), _Handler)
    thread = threading.Thread(target=server.serve_forever, daemon=True)
    thread.start()
    try:
        host, port = server.server_address[:2]
        yield f"http://{host!s}:{port}"
    finally:
        server.shutdown()
        server.server_close()
        thread.join()


def test_get_json_returns_decoded_body(base_url: str) -> None:
    result = get_json(f"{base_url}/ok")
    assert result.status == 200
    assert result.data == {"ok": True}
    assert result.headers["content-type"] == "application/json"


def test_get_json_raises_on_error_status(base_url: str) -> None:
    with pytest.raises(UpstreamStatusError) as excinfo:
        get_json(f"{base_url}/missing")
    assert excinfo.value.status == 404


@pytest.mark.parametrize("url", ["", "ftp://example.com/file", "/relative/path"])
def test_get_json_rejects_unsupported_urls(url: str) -> None:
    with pytest.raises(InvalidUrlError):
        get_json(url)


def test_get_json_rejects_non_positive_timeout() -> None:
    with pytest.raises(ValueError, match="timeout"):
        get_json("http://example.com", timeout=0)
```

---
## Common mistakes

- Using GET for actions that change data. Crawlers, prefetchers, and browsers may call GET URLs freely.
- Treating the fragment (`#section`) as something the server can read. It never leaves the browser.
- Confusing 401 and 403, or returning 200 with an error message in the body, which breaks clients and monitoring.
- Assuming HTTPS means the application is safe. It protects transport only.
- Calling external services without a timeout. A hung dependency then hangs your worker indefinitely.
- Catching `URLError` before `HTTPError`. `HTTPError` is a subclass of `URLError`, so the broader handler would swallow it.
- Forgetting `Content-Length` or `Content-Type` in hand-written responses and wondering why clients hang or misrender.

---
## Practice exercises

1. Run the `http.server` example and use `curl -v` to identify the request line, three request headers, the status line, and two response headers.
2. Extend `HelloHandler` so that `GET /greet?name=Ada` returns `Hello, Ada`, and a missing `name` returns a 400 response with a plain-text explanation.
3. Add a `do_POST` method that reads `Content-Length` bytes from `self.rfile` and echoes the body back. Test it with `curl -d`.
4. Write a function `describe_status(code: int) -> str` that returns the family name (`"success"`, `"client error"`, and so on) and raises `ValueError` for codes outside 100 to 599. Add parametrized tests.
5. Use the browser Network tab on any site you use. Find one redirect, one cached response (304 or "from disk cache"), and one `Set-Cookie` header. Record what each tells you.
6. Write a function that takes a URL and returns a dataclass with scheme, host, port (with defaults filled in for http and https), path, and a query dictionary. Reject URLs without a scheme.
7. Modify `get_json` to retry up to three times on `TransportError` and on 502, 503, and 504 responses, with a short delay between attempts, and never retry on other 4xx errors. Write tests for each branch.

---
## Summary

HTTP is a stateless request and response protocol. A URL names a resource, a method states intent, headers carry metadata, and a status code reports the outcome. Safe and idempotent methods let clients and caches behave predictably, and cookies are the main way state is layered on top. TLS encrypts the channel but does not secure the application. A framework exists to take over the repetitive mechanics so your code deals only with application logic. Whenever your own code makes a request, use timeouts, validate inputs, raise explicit errors, and test with a real or faked server.

---