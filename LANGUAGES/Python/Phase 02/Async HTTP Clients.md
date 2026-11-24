```
Your backend rarely works alone.
It calls other services constantly:

  Your API ──► Payment gateway (Stripe)
  Your API ──► Email service (SendGrid)
  Your API ──► Authentication (Auth0)
  Your API ──► Maps (Google Maps)
  Your API ──► Your own microservices
  Your API ──► Third-party data APIs

The WRONG way:
  import requests
  response = requests.get("https://api.stripe.com/...")
  
  In an async FastAPI endpoint, this BLOCKS the event loop.
  While waiting for Stripe (200ms), your server can't
  handle any other requests. With 100 concurrent users
  all waiting for Stripe → server appears frozen.

The RIGHT way:
  async with httpx.AsyncClient() as client:
      response = await client.get("https://api.stripe.com/...")
  
  Yields control while waiting. Other requests process normally.
  
This phase: master async HTTP clients for production backends.
```

---

## Setup

Bash

```
cd ~/projects
mkdir async_http_clients
cd async_http_clients
python3 -m venv venv
source venv/bin/activate

pip install httpx aiohttp fastapi uvicorn \
    pydantic pydantic-settings python-dotenv \
    tenacity backoff

touch clients.py
code .
```

---

## 1. 🔍 httpx — The Modern Choice

### Why httpx?

text

```
httpx = HTTP client for Python with async support.

httpx vs requests:
  requests:   sync only, widely used
  httpx:      sync AND async, same API feel
              modern, actively maintained
              built-in timeout control
              built-in retry support
              HTTP/2 support
              type hints throughout

httpx vs aiohttp:
  aiohttp:    async only, older, more complex
  httpx:      sync + async, simpler API
              better test support
              more Pythonic

In production Python backends today:
  httpx = gold standard for HTTP clients
```

Python

```
# clients.py
import asyncio
import httpx
import time
from typing import Any, Dict, Optional, List


# ─────────────────────────────────────────
# BASIC USAGE
# ─────────────────────────────────────────
async def httpx_basics():
    """Core httpx operations."""
    print("=== httpx Basics ===\n")

    # ─── Simple GET ───────────────────────────────────────────
    async with httpx.AsyncClient() as client:
        response = await client.get("https://httpbin.org/get")

        print(f"Status:      {response.status_code}")
        print(f"URL:         {response.url}")
        print(f"Headers:     {dict(response.headers)}")
        print(f"Content-Type:{response.headers.get('content-type')}")
        print(f"Body (JSON): {response.json()['url']}")
        print(f"Elapsed:     {response.elapsed.total_seconds()*1000:.0f}ms")

    # ─── Query Parameters ─────────────────────────────────────
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://httpbin.org/get",
            params={               # auto URL-encoded
                "search": "python async",
                "page": 1,
                "limit": 20,
                "filter": ["active", "verified"],  # list = repeated param
            }
        )
        print(f"\nURL with params: {response.url}")
        # → https://httpbin.org/get?search=python+async&page=1&limit=20&filter=active&filter=verified

    # ─── POST with JSON body ─────────────────────────────────
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://httpbin.org/post",
            json={                  # sets Content-Type: application/json
                "name": "Alice",
                "email": "alice@test.com",
                "role": "admin"
            }
        )
        data = response.json()
        print(f"\nPOST response: {data['json']}")

    # ─── Headers ─────────────────────────────────────────────
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://httpbin.org/headers",
            headers={
                "Authorization": "Bearer my-jwt-token",
                "X-API-Key": "secret-api-key",
                "X-Request-ID": "abc-123",
                "Accept": "application/json",
                "User-Agent": "MyApp/1.0 (Python/3.12)",
            }
        )
        print(f"\nSent headers: {response.json()['headers']}")

    # ─── Form data and file uploads ───────────────────────────
    async with httpx.AsyncClient() as client:
        # Form data
        response = await client.post(
            "https://httpbin.org/post",
            data={                  # URL-encoded form
                "username": "alice",
                "password": "secret"
            }
        )

        # File upload (multipart/form-data)
        files = {
            "file": ("profile.jpg", b"fake_image_data", "image/jpeg"),
            "document": ("resume.pdf", b"fake_pdf_data", "application/pdf"),
        }
        response = await client.post(
            "https://httpbin.org/post",
            files=files,
            data={"user_id": "42"}  # additional form fields
        )

    # ─── Different HTTP methods ───────────────────────────────
    async with httpx.AsyncClient() as client:
        base = "https://httpbin.org"

        await client.put(f"{base}/put", json={"key": "value"})
        await client.patch(f"{base}/patch", json={"field": "updated"})
        await client.delete(f"{base}/delete")

        # HEAD — get headers without body
        head_response = await client.head(f"{base}/get")
        print(f"\nHEAD headers: Content-Type={head_response.headers.get('content-type')}")


asyncio.run(httpx_basics())
```

### Response Handling

Python

```
async def response_handling():
    """Handle different response types."""
    print("\n=== Response Handling ===\n")

    async with httpx.AsyncClient() as client:

        # ─── JSON response ─────────────────────────────────────
        response = await client.get("https://httpbin.org/json")

        # Different ways to get JSON
        data = response.json()                    # dict
        text = response.text                      # string
        raw = response.content                    # bytes

        print(f"JSON: {type(data)}")
        print(f"Text: {type(text)}")
        print(f"Raw:  {type(raw)}")

        # ─── Check status ─────────────────────────────────────
        response = await client.get("https://httpbin.org/status/200")
        print(f"\nIs success: {response.is_success}")        # 2xx
        print(f"Is redirect: {response.is_redirect}")        # 3xx
        print(f"Is error: {response.is_error}")              # 4xx or 5xx
        print(f"Is client error: {response.is_client_error}")# 4xx
        print(f"Is server error: {response.is_server_error}")# 5xx

        # Raise exception for error status codes
        try:
            response = await client.get("https://httpbin.org/status/404")
            response.raise_for_status()   # raises HTTPStatusError
        except httpx.HTTPStatusError as e:
            print(f"\nHTTP Error: {e.response.status_code} - {e.request.url}")

        # ─── Streaming large responses ─────────────────────────
        # For large files, don't load everything into memory
        async with client.stream("GET", "https://httpbin.org/bytes/1024") as stream:
            print(f"\nStreaming response:")
            chunk_count = 0
            async for chunk in stream.aiter_bytes(chunk_size=256):
                chunk_count += 1
                # Process each chunk — never loads whole file
            print(f"  Downloaded in {chunk_count} chunks")

        # ─── Async iteration over lines ──────────────────────
        async with client.stream("GET", "https://httpbin.org/stream/5") as stream:
            print("\nLine by line:")
            async for line in stream.aiter_lines():
                if line.strip():
                    print(f"  Line: {line[:50]}...")


asyncio.run(response_handling())
```

---

## 2. ⚙️ Client Configuration — Production Setup

Python

```
import httpx
import asyncio
from typing import Optional


class HTTPClientConfig:
    """
    Production-ready HTTP client configuration.
    Create ONE client and reuse it — don't create per-request.
    """

    def __init__(
        self,
        base_url: str = "",
        timeout: float = 30.0,
        max_connections: int = 100,
        max_keepalive_connections: int = 20,
        api_key: Optional[str] = None,
        bearer_token: Optional[str] = None,
        verify_ssl: bool = True,
        follow_redirects: bool = True,
    ):
        # ─── Timeouts ─────────────────────────────────────────
        # Different timeouts for different phases of the request
        timeout_config = httpx.Timeout(
            connect=5.0,       # connecting to host
            read=timeout,      # reading response
            write=10.0,        # writing request body
            pool=5.0,          # waiting for connection from pool
        )

        # ─── Connection Pool ─────────────────────────────────
        limits = httpx.Limits(
            max_connections=max_connections,
            max_keepalive_connections=max_keepalive_connections,
            keepalive_expiry=30,   # seconds to keep idle connections
        )

        # ─── Default Headers ─────────────────────────────────
        headers = {
            "User-Agent": "MyApp/1.0 Python/3.12",
            "Accept": "application/json",
            "Accept-Encoding": "gzip, deflate, br",
        }

        if api_key:
            headers["X-API-Key"] = api_key

        if bearer_token:
            headers["Authorization"] = f"Bearer {bearer_token}"

        self.client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout_config,
            limits=limits,
            headers=headers,
            verify=verify_ssl,
            follow_redirects=follow_redirects,
            http2=True,           # enable HTTP/2 when available
        )

    async def __aenter__(self):
        return self.client

    async def __aexit__(self, *args):
        await self.client.aclose()


# ─────────────────────────────────────────
# SINGLETON CLIENT — reuse across requests
# ─────────────────────────────────────────
class SingletonHTTPClient:
    """
    Single HTTP client shared across all requests.
    Created at startup, closed at shutdown.

    This is the production pattern:
    - Connection pool is reused
    - No overhead of creating new client per request
    - Proper lifecycle management
    """
    _instance: Optional[httpx.AsyncClient] = None

    @classmethod
    async def get_client(cls) -> httpx.AsyncClient:
        if cls._instance is None:
            cls._instance = httpx.AsyncClient(
                timeout=httpx.Timeout(connect=5.0, read=30.0),
                limits=httpx.Limits(
                    max_connections=100,
                    max_keepalive_connections=20
                ),
                headers={"User-Agent": "MyApp/1.0"},
                follow_redirects=True,
            )
        return cls._instance

    @classmethod
    async def close(cls):
        if cls._instance:
            await cls._instance.aclose()
            cls._instance = None


# FastAPI integration
from fastapi import FastAPI
app = FastAPI()

_http_client: Optional[httpx.AsyncClient] = None


@app.on_event("startup")
async def create_http_client():
    global _http_client
    _http_client = httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=30.0),
        limits=httpx.Limits(
            max_connections=100,
            max_keepalive_connections=20
        ),
    )


@app.on_event("shutdown")
async def close_http_client():
    if _http_client:
        await _http_client.aclose()


def get_http_client() -> httpx.AsyncClient:
    """FastAPI dependency to get the shared client."""
    return _http_client
```

---

## 3. 🔄 Retry Logic & Error Handling

Python

```
import httpx
import asyncio
import time
from typing import Optional, Callable, Any
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log,
)
import logging

logger = logging.getLogger(__name__)


# ─────────────────────────────────────────
# MANUAL RETRY WITH EXPONENTIAL BACKOFF
# ─────────────────────────────────────────
async def fetch_with_retry(
    client: httpx.AsyncClient,
    url: str,
    method: str = "GET",
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    backoff_factor: float = 2.0,
    **kwargs
) -> httpx.Response:
    """
    Fetch URL with exponential backoff retry.

    Retry on:
      - Connection errors (server unreachable)
      - Timeout errors (server too slow)
      - 429 Too Many Requests (with Retry-After respect)
      - 5xx Server Errors (server having issues)

    Don't retry on:
      - 4xx Client Errors (our fault, fix the request)
      - 200-399 Success responses
    """
    last_error = None

    for attempt in range(1, max_attempts + 1):
        try:
            response = await client.request(method, url, **kwargs)

            # Success
            if response.is_success:
                return response

            # Respect rate limit header
            if response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", base_delay))
                logger.warning(
                    f"Rate limited. Waiting {retry_after}s",
                    extra={"url": url, "attempt": attempt}
                )
                if attempt < max_attempts:
                    await asyncio.sleep(retry_after)
                    continue

            # Don't retry client errors (4xx except 429)
            if response.is_client_error:
                response.raise_for_status()

            # Server errors — retry
            if response.is_server_error:
                last_error = httpx.HTTPStatusError(
                    f"Server error {response.status_code}",
                    request=response.request,
                    response=response
                )

        except (httpx.ConnectError, httpx.ConnectTimeout) as e:
            last_error = e
            logger.warning(
                f"Connection failed (attempt {attempt}/{max_attempts}): {e}",
                extra={"url": url}
            )

        except httpx.TimeoutException as e:
            last_error = e
            logger.warning(
                f"Timeout (attempt {attempt}/{max_attempts})",
                extra={"url": url}
            )

        except httpx.HTTPStatusError:
            raise   # don't retry client errors

        # Calculate delay before retry
        if attempt < max_attempts:
            delay = min(
                base_delay * (backoff_factor ** (attempt - 1)),
                max_delay
            )
            # Add jitter to prevent thundering herd
            import random
            jitter = random.uniform(0, delay * 0.1)
            actual_delay = delay + jitter

            logger.info(f"Retrying in {actual_delay:.2f}s...")
            await asyncio.sleep(actual_delay)

    raise last_error or Exception(f"All {max_attempts} attempts failed")


# ─────────────────────────────────────────
# USING TENACITY LIBRARY (production standard)
# ─────────────────────────────────────────
def is_retryable_error(exc: Exception) -> bool:
    """Determine if we should retry this error."""
    if isinstance(exc, httpx.ConnectError):
        return True
    if isinstance(exc, httpx.TimeoutException):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in {429, 500, 502, 503, 504}
    return False


@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((
        httpx.ConnectError,
        httpx.TimeoutException,
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def resilient_fetch(
    client: httpx.AsyncClient,
    url: str
) -> dict:
    """Fetch with automatic retry using tenacity."""
    response = await client.get(url)
    response.raise_for_status()
    return response.json()


# ─────────────────────────────────────────
# CIRCUIT BREAKER PATTERN
# ─────────────────────────────────────────
import enum
from datetime import datetime, timedelta


class CircuitState(enum.Enum):
    CLOSED = "closed"       # normal, requests pass through
    OPEN = "open"           # failing, reject requests immediately
    HALF_OPEN = "half_open" # testing if recovered


class CircuitBreaker:
    """
    Circuit breaker pattern for HTTP clients.

    Prevents cascading failures:
    If service is down, stop sending requests immediately
    instead of waiting for timeout every time.

    States:
      CLOSED:    Normal operation. Track failures.
      OPEN:      Service is down. Reject all requests.
      HALF_OPEN: Test recovery. Allow one request through.
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        name: str = "circuit"
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.name = name

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.success_count = 0

    def _should_attempt(self) -> bool:
        """Check if request should be attempted."""
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Check if recovery timeout has passed
            if (datetime.now() - self.last_failure_time >
                    timedelta(seconds=self.recovery_timeout)):
                self.state = CircuitState.HALF_OPEN
                logger.info(f"Circuit {self.name}: OPEN → HALF_OPEN")
                return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            return True

        return False

    def _on_success(self):
        """Record successful request."""
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
            self.failure_count = 0
            logger.info(f"Circuit {self.name}: HALF_OPEN → CLOSED (recovered)")
        self.success_count += 1

    def _on_failure(self):
        """Record failed request."""
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.error(
                f"Circuit {self.name}: CLOSED → OPEN "
                f"({self.failure_count} failures)"
            )

    async def call(self, coro):
        """Execute coroutine through circuit breaker."""
        if not self._should_attempt():
            raise Exception(
                f"Circuit breaker OPEN for {self.name}. "
                f"Service unavailable."
            )
        try:
            result = await coro
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    @property
    def status(self) -> dict:
        return {
            "name": self.name,
            "state": self.state.value,
            "failure_count": self.failure_count,
            "success_count": self.success_count,
        }


# ─────────────────────────────────────────
# DEMO
# ─────────────────────────────────────────
async def retry_demo():
    print("\n=== Retry & Circuit Breaker Demo ===\n")

    async with httpx.AsyncClient(timeout=10.0) as client:
        # Test retry
        print("1. Testing retry logic:")
        try:
            response = await fetch_with_retry(
                client,
                "https://httpbin.org/status/200",
                max_attempts=3
            )
            print(f"   Success: {response.status_code}")
        except Exception as e:
            print(f"   Failed: {e}")

        # Test circuit breaker
        print("\n2. Testing circuit breaker:")
        breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=10)

        # Simulate failures to trip circuit
        for i in range(4):
            try:
                await breaker.call(
                    client.get("https://httpbin.org/status/500")
                )
            except Exception as e:
                print(f"   Attempt {i+1}: {type(e).__name__}")

        print(f"   Circuit status: {breaker.status}")


asyncio.run(retry_demo())
```

---

## 4. 🏗️ Building API Client Classes

Python

```
import httpx
import asyncio
from typing import Optional, Dict, Any, List
from dataclasses import dataclass
from pydantic import BaseModel


# ─────────────────────────────────────────
# BASE API CLIENT
# ─────────────────────────────────────────
class BaseAPIClient:
    """
    Base class for all external API clients.
    Handles: auth, retries, error handling, logging.
    """

    BASE_URL: str = ""
    TIMEOUT: float = 30.0

    def __init__(
        self,
        api_key: Optional[str] = None,
        bearer_token: Optional[str] = None,
        base_url: Optional[str] = None,
    ):
        headers = {
            "User-Agent": "MyApp/1.0",
            "Accept": "application/json",
        }

        if api_key:
            headers["X-API-Key"] = api_key
        if bearer_token:
            headers["Authorization"] = f"Bearer {bearer_token}"

        self._client = httpx.AsyncClient(
            base_url=base_url or self.BASE_URL,
            headers=headers,
            timeout=httpx.Timeout(
                connect=5.0,
                read=self.TIMEOUT,
                write=10.0,
            ),
            limits=httpx.Limits(
                max_connections=50,
                max_keepalive_connections=10,
            ),
            follow_redirects=True,
        )

    async def _request(
        self,
        method: str,
        path: str,
        **kwargs
    ) -> httpx.Response:
        """
        Make an HTTP request with error handling.
        Override in subclasses for custom behavior.
        """
        try:
            response = await self._client.request(method, path, **kwargs)
            response.raise_for_status()
            return response

        except httpx.HTTPStatusError as e:
            status = e.response.status_code
            try:
                error_body = e.response.json()
            except Exception:
                error_body = {"message": e.response.text}

            logger.error(
                f"API error: {status} {method} {path}",
                extra={"error_body": error_body}
            )
            raise APIError(
                status_code=status,
                message=error_body.get("message", "API request failed"),
                response=error_body
            ) from e

        except httpx.TimeoutException:
            logger.error(f"Timeout: {method} {path}")
            raise APIError(503, "External service timeout")

        except httpx.ConnectError:
            logger.error(f"Connection failed: {method} {path}")
            raise APIError(503, "Cannot connect to external service")

    async def get(self, path: str, **kwargs) -> Dict:
        response = await self._request("GET", path, **kwargs)
        return response.json()

    async def post(self, path: str, **kwargs) -> Dict:
        response = await self._request("POST", path, **kwargs)
        return response.json()

    async def put(self, path: str, **kwargs) -> Dict:
        response = await self._request("PUT", path, **kwargs)
        return response.json()

    async def patch(self, path: str, **kwargs) -> Dict:
        response = await self._request("PATCH", path, **kwargs)
        return response.json()

    async def delete(self, path: str, **kwargs) -> None:
        await self._request("DELETE", path, **kwargs)

    async def close(self):
        await self._client.aclose()

    async def __aenter__(self):
        return self

    async def __aexit__(self, *args):
        await self.close()


class APIError(Exception):
    """Custom exception for API client errors."""

    def __init__(
        self,
        status_code: int,
        message: str,
        response: Optional[Dict] = None
    ):
        super().__init__(message)
        self.status_code = status_code
        self.message = message
        self.response = response or {}

    def __repr__(self):
        return f"APIError({self.status_code}: {self.message})"


# ─────────────────────────────────────────
# GITHUB API CLIENT — real example
# ─────────────────────────────────────────
class GitHubClient(BaseAPIClient):
    """
    GitHub API v3 client.
    Shows real-world API client implementation.
    """

    BASE_URL = "https://api.github.com"
    TIMEOUT = 30.0

    def __init__(self, token: Optional[str] = None):
        super().__init__(bearer_token=token)

    # ─── User endpoints ────────────────────────────────────────
    async def get_user(self, username: str) -> Dict:
        """Get public GitHub user profile."""
        return await self.get(f"/users/{username}")

    async def get_authenticated_user(self) -> Dict:
        """Get authenticated user's profile."""
        return await self.get("/user")

    async def search_users(
        self,
        query: str,
        sort: str = "followers",
        order: str = "desc",
        per_page: int = 10,
        page: int = 1
    ) -> Dict:
        """Search GitHub users."""
        return await self.get(
            "/search/users",
            params={
                "q": query,
                "sort": sort,
                "order": order,
                "per_page": per_page,
                "page": page
            }
        )

    # ─── Repository endpoints ──────────────────────────────────
    async def get_repo(self, owner: str, repo: str) -> Dict:
        """Get repository details."""
        return await self.get(f"/repos/{owner}/{repo}")

    async def list_user_repos(
        self,
        username: str,
        type: str = "public",
        sort: str = "updated",
        per_page: int = 30
    ) -> List[Dict]:
        """List user's repositories."""
        return await self.get(
            f"/users/{username}/repos",
            params={
                "type": type,
                "sort": sort,
                "per_page": per_page
            }
        )

    async def get_repo_languages(self, owner: str, repo: str) -> Dict:
        """Get programming languages used in a repo."""
        return await self.get(f"/repos/{owner}/{repo}/languages")

    async def list_contributors(
        self,
        owner: str,
        repo: str,
        per_page: int = 30
    ) -> List[Dict]:
        """List repository contributors."""
        return await self.get(
            f"/repos/{owner}/{repo}/contributors",
            params={"per_page": per_page}
        )

    # ─── Concurrent operations ────────────────────────────────
    async def get_user_with_repos(self, username: str) -> Dict:
        """Fetch user profile and repos concurrently."""
        user, repos = await asyncio.gather(
            self.get_user(username),
            self.list_user_repos(username),
            return_exceptions=True   # don't fail if one fails
        )

        return {
            "user": user if not isinstance(user, Exception) else None,
            "repos": repos if not isinstance(repos, Exception) else [],
            "repo_count": len(repos) if isinstance(repos, list) else 0,
        }

    async def analyze_repo(self, owner: str, repo: str) -> Dict:
        """Get comprehensive repo info with concurrent requests."""
        repo_info, languages, contributors = await asyncio.gather(
            self.get_repo(owner, repo),
            self.get_repo_languages(owner, repo),
            self.list_contributors(owner, repo, per_page=5),
            return_exceptions=True
        )

        return {
            "repo": repo_info,
            "languages": languages,
            "top_contributors": contributors[:5] if isinstance(contributors, list) else [],
        }


# Test GitHub client
async def test_github():
    print("\n=== GitHub API Client ===\n")

    async with GitHubClient() as client:
        # Get Python language creator's profile
        user = await client.get_user("gvanrossum")
        print(f"Name: {user['name']}")
        print(f"Followers: {user['followers']:,}")
        print(f"Public repos: {user['public_repos']}")

        # Get repo info concurrently
        repo_data = await client.analyze_repo("tiangolo", "fastapi")
        print(f"\nFastAPI:")
        print(f"  Stars: {repo_data['repo']['stargazers_count']:,}")
        print(f"  Languages: {list(repo_data['languages'].keys())}")


asyncio.run(test_github())
```

---

## 5. 🔗 Webhook Client

Python

```
import asyncio
import hmac
import hashlib
import json
import time
from typing import Dict, Any, Optional, Callable, List
import httpx


# ─────────────────────────────────────────
# WEBHOOK SENDER — sending webhooks to clients
# ─────────────────────────────────────────
class WebhookSender:
    """
    Send webhook events to external URLs.
    Used when YOUR service needs to notify others of events.

    Example: When a payment is processed, notify the merchant's system.
    """

    def __init__(
        self,
        signing_secret: Optional[str] = None,
        timeout: float = 30.0,
        max_retries: int = 3,
    ):
        self.signing_secret = signing_secret
        self.max_retries = max_retries
        self._client = httpx.AsyncClient(
            timeout=httpx.Timeout(connect=5.0, read=timeout),
            headers={
                "Content-Type": "application/json",
                "User-Agent": "MyApp-Webhooks/1.0",
            }
        )

    def _generate_signature(self, payload: str, timestamp: int) -> str:
        """
        Generate HMAC signature for webhook verification.
        Recipient can verify this to confirm the webhook is genuine.
        """
        if not self.signing_secret:
            return ""

        signed_payload = f"{timestamp}.{payload}"
        signature = hmac.new(
            self.signing_secret.encode(),
            signed_payload.encode(),
            hashlib.sha256
        ).hexdigest()
        return f"v1={signature}"

    async def send(
        self,
        url: str,
        event_type: str,
        payload: Dict[str, Any],
        headers: Optional[Dict] = None,
    ) -> Dict[str, Any]:
        """
        Send a webhook event to a URL.
        Returns delivery result with status and timing.
        """
        timestamp = int(time.time())
        body = json.dumps({
            "event": event_type,
            "timestamp": timestamp,
            "data": payload,
        })

        request_headers = {
            "X-Webhook-Event": event_type,
            "X-Webhook-Timestamp": str(timestamp),
            "X-Webhook-ID": f"wh_{timestamp}_{hash(url) % 10000:04d}",
        }

        if self.signing_secret:
            request_headers["X-Webhook-Signature"] = (
                self._generate_signature(body, timestamp)
            )

        if headers:
            request_headers.update(headers)

        last_error = None
        for attempt in range(1, self.max_retries + 1):
            try:
                start = time.perf_counter()
                response = await self._client.post(
                    url,
                    content=body,
                    headers=request_headers,
                )
                elapsed_ms = (time.perf_counter() - start) * 1000

                if response.is_success:
                    return {
                        "success": True,
                        "url": url,
                        "event": event_type,
                        "status_code": response.status_code,
                        "attempt": attempt,
                        "elapsed_ms": elapsed_ms,
                    }

                # Server error — retry
                if response.is_server_error and attempt < self.max_retries:
                    await asyncio.sleep(2 ** attempt)  # exponential backoff
                    continue

                return {
                    "success": False,
                    "url": url,
                    "event": event_type,
                    "status_code": response.status_code,
                    "error": f"HTTP {response.status_code}",
                    "attempt": attempt,
                }

            except (httpx.TimeoutException, httpx.ConnectError) as e:
                last_error = str(e)
                if attempt < self.max_retries:
                    await asyncio.sleep(2 ** attempt)

        return {
            "success": False,
            "url": url,
            "event": event_type,
            "error": last_error,
            "attempt": self.max_retries,
        }

    async def send_batch(
        self,
        deliveries: List[Dict[str, Any]],
        max_concurrent: int = 10,
    ) -> List[Dict]:
        """Send multiple webhooks concurrently with rate limiting."""
        semaphore = asyncio.Semaphore(max_concurrent)

        async def send_one(delivery: Dict) -> Dict:
            async with semaphore:
                return await self.send(
                    url=delivery["url"],
                    event_type=delivery["event_type"],
                    payload=delivery["payload"],
                )

        return await asyncio.gather(*[
            send_one(d) for d in deliveries
        ])

    async def close(self):
        await self._client.aclose()


# ─────────────────────────────────────────
# WEBHOOK RECEIVER — verifying incoming webhooks
# ─────────────────────────────────────────
class WebhookVerifier:
    """
    Verify incoming webhook signatures.
    Used when OTHER services send webhooks TO you.
    """

    @staticmethod
    def verify_signature(
        payload: bytes,
        signature: str,
        secret: str,
        timestamp: Optional[int] = None,
        tolerance_seconds: int = 300  # 5 minutes
    ) -> bool:
        """
        Verify HMAC webhook signature.
        Protects against:
          - Forged webhooks (signature mismatch)
          - Replay attacks (timestamp too old)
        """
        if timestamp:
            # Check timestamp freshness (prevent replay attacks)
            age = abs(int(time.time()) - timestamp)
            if age > tolerance_seconds:
                return False

        # Compute expected signature
        signed_payload = (
            f"{timestamp}.{payload.decode()}"
            if timestamp
            else payload.decode()
        )

        expected = hmac.new(
            secret.encode(),
            signed_payload.encode(),
            hashlib.sha256
        ).hexdigest()

        # Extract signature value (handle "v1=..." prefix)
        actual = signature
        if "=" in signature:
            actual = signature.split("=", 1)[1]

        # Constant-time comparison (prevents timing attacks)
        return hmac.compare_digest(expected, actual)


# FastAPI webhook endpoints
from fastapi import FastAPI, Request, HTTPException, Header
from typing import Optional

webhook_app = FastAPI()
WEBHOOK_SECRET = "my-webhook-secret"
webhook_sender = WebhookSender(signing_secret=WEBHOOK_SECRET)


@webhook_app.post("/webhooks/receive")
async def receive_webhook(
    request: Request,
    x_webhook_signature: Optional[str] = Header(None),
    x_webhook_timestamp: Optional[str] = Header(None),
):
    """Receive and verify an incoming webhook."""
    body = await request.body()

    # Verify signature
    if x_webhook_signature:
        timestamp = int(x_webhook_timestamp) if x_webhook_timestamp else None
        is_valid = WebhookVerifier.verify_signature(
            payload=body,
            signature=x_webhook_signature,
            secret=WEBHOOK_SECRET,
            timestamp=timestamp
        )

        if not is_valid:
            raise HTTPException(status_code=401, detail="Invalid webhook signature")

    # Process the webhook
    data = await request.json()
    event_type = data.get("event")

    print(f"Received webhook: {event_type}")
    print(f"Payload: {data.get('data')}")

    # Always return 200 quickly (process async if needed)
    return {"received": True}


# Test webhook system
async def webhook_demo():
    print("\n=== Webhook Demo ===\n")

    sender = WebhookSender(
        signing_secret="test-secret",
        max_retries=2
    )

    # Send a single webhook (to httpbin which echoes it back)
    result = await sender.send(
        url="https://httpbin.org/post",
        event_type="payment.completed",
        payload={
            "payment_id": "pay_123",
            "amount": 99.99,
            "currency": "USD",
            "customer_id": "cust_456"
        }
    )
    print(f"Webhook sent: {result}")

    # Send batch of webhooks
    deliveries = [
        {
            "url": "https://httpbin.org/post",
            "event_type": f"event.{i}",
            "payload": {"index": i}
        }
        for i in range(5)
    ]

    start = time.perf_counter()
    results = await sender.send_batch(deliveries, max_concurrent=3)
    elapsed = time.perf_counter() - start

    successful = sum(1 for r in results if r.get("success"))
    print(f"\nBatch: {successful}/{len(results)} successful in {elapsed:.2f}s")

    await sender.close()


asyncio.run(webhook_demo())
```

---

## 6. 🔌 Real-World API Integrations

Python

```
import asyncio
import httpx
from typing import Optional, Dict, Any, List
from pydantic import BaseModel


# ─────────────────────────────────────────
# STRIPE PAYMENT CLIENT
# ─────────────────────────────────────────
class StripeClient(BaseAPIClient):
    """
    Stripe payment API client.
    Production pattern for payment integrations.
    """

    BASE_URL = "https://api.stripe.com/v1"
    TIMEOUT = 30.0

    def __init__(self, secret_key: str):
        # Stripe uses HTTP Basic Auth with API key as username
        self._client = httpx.AsyncClient(
            base_url=self.BASE_URL,
            auth=(secret_key, ""),   # (username, password) — password empty
            headers={
                "Stripe-Version": "2024-06-20",
                "Content-Type": "application/x-www-form-urlencoded",
                "User-Agent": "MyApp/1.0"
            },
            timeout=httpx.Timeout(connect=5.0, read=30.0)
        )

    async def create_payment_intent(
        self,
        amount: int,           # in cents
        currency: str,
        customer_id: Optional[str] = None,
        metadata: Optional[Dict] = None,
    ) -> Dict:
        """Create a Stripe PaymentIntent."""
        data = {
            "amount": amount,
            "currency": currency.lower(),
            "automatic_payment_methods[enabled]": "true",
        }
        if customer_id:
            data["customer"] = customer_id
        if metadata:
            for key, value in metadata.items():
                data[f"metadata[{key}]"] = value

        response = await self._client.post("/payment_intents", data=data)
        response.raise_for_status()
        return response.json()

    async def retrieve_payment_intent(self, payment_intent_id: str) -> Dict:
        """Get PaymentIntent status."""
        response = await self._client.get(
            f"/payment_intents/{payment_intent_id}"
        )
        response.raise_for_status()
        return response.json()

    async def create_customer(
        self,
        email: str,
        name: Optional[str] = None,
        metadata: Optional[Dict] = None,
    ) -> Dict:
        """Create a Stripe Customer."""
        data = {"email": email}
        if name:
            data["name"] = name
        if metadata:
            for key, value in metadata.items():
                data[f"metadata[{key}]"] = value

        response = await self._client.post("/customers", data=data)
        response.raise_for_status()
        return response.json()

    async def list_payment_methods(
        self,
        customer_id: str,
        type: str = "card"
    ) -> List[Dict]:
        """List customer's payment methods."""
        response = await self._client.get(
            "/payment_methods",
            params={"customer": customer_id, "type": type}
        )
        response.raise_for_status()
        return response.json()["data"]


# ─────────────────────────────────────────
# SENDGRID EMAIL CLIENT
# ─────────────────────────────────────────
class SendGridClient(BaseAPIClient):
    """SendGrid email API client."""

    BASE_URL = "https://api.sendgrid.com/v3"
    TIMEOUT = 15.0

    def __init__(self, api_key: str, from_email: str, from_name: str = ""):
        super().__init__(bearer_token=api_key)
        self.from_email = from_email
        self.from_name = from_name

    async def send_email(
        self,
        to_email: str,
        to_name: str = "",
        subject: str = "",
        html_content: str = "",
        text_content: str = "",
        template_id: Optional[str] = None,
        template_data: Optional[Dict] = None,
    ) -> Dict:
        """Send a single email via SendGrid."""
        payload = {
            "personalizations": [{
                "to": [{"email": to_email, "name": to_name}],
            }],
            "from": {
                "email": self.from_email,
                "name": self.from_name
            },
            "subject": subject,
        }

        if template_id:
            payload["template_id"] = template_id
            if template_data:
                payload["personalizations"][0]["dynamic_template_data"] = (
                    template_data
                )
        else:
            payload["content"] = []
            if text_content:
                payload["content"].append({
                    "type": "text/plain",
                    "value": text_content
                })
            if html_content:
                payload["content"].append({
                    "type": "text/html",
                    "value": html_content
                })

        return await self.post("/mail/send", json=payload)

    async def send_bulk(
        self,
        emails: List[Dict[str, str]],
        subject: str,
        html_content: str,
        max_concurrent: int = 5,
    ) -> List[Dict]:
        """Send bulk emails with rate limiting."""
        semaphore = asyncio.Semaphore(max_concurrent)

        async def send_one(email_data: Dict) -> Dict:
            async with semaphore:
                try:
                    await self.send_email(
                        to_email=email_data["email"],
                        to_name=email_data.get("name", ""),
                        subject=subject,
                        html_content=html_content
                    )
                    return {"email": email_data["email"], "status": "sent"}
                except Exception as e:
                    return {
                        "email": email_data["email"],
                        "status": "failed",
                        "error": str(e)
                    }

        return await asyncio.gather(*[send_one(e) for e in emails])


# ─────────────────────────────────────────
# OPENAI CLIENT — Streaming
# ─────────────────────────────────────────
class OpenAIClient(BaseAPIClient):
    """OpenAI API client with streaming support."""

    BASE_URL = "https://api.openai.com/v1"
    TIMEOUT = 120.0   # AI responses can take time

    def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
        super().__init__(bearer_token=api_key)
        self.model = model

    async def chat_completion(
        self,
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
        max_tokens: int = 1000,
    ) -> str:
        """Non-streaming chat completion."""
        data = await self.post(
            "/chat/completions",
            json={
                "model": self.model,
                "messages": messages,
                "temperature": temperature,
                "max_tokens": max_tokens,
            }
        )
        return data["choices"][0]["message"]["content"]

    async def stream_chat_completion(
        self,
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
    ):
        """
        Streaming chat completion.
        Yields tokens as they arrive from OpenAI.
        Used for real-time UI updates.
        """
        async with self._client.stream(
            "POST",
            "/chat/completions",
            json={
                "model": self.model,
                "messages": messages,
                "temperature": temperature,
                "stream": True,     # enable streaming
            }
        ) as response:
            response.raise_for_status()

            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = line[6:]   # remove "data: " prefix
                    if data == "[DONE]":
                        break
                    try:
                        import json
                        chunk = json.loads(data)
                        token = chunk["choices"][0]["delta"].get("content", "")
                        if token:
                            yield token
                    except json.JSONDecodeError:
                        continue


# FastAPI streaming endpoint
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

api = FastAPI()


@api.post("/ai/chat/stream")
async def stream_response(messages: List[Dict]) -> StreamingResponse:
    """
    Stream AI responses to the client.
    Text appears word by word as it's generated.
    """
    # Real: openai_client = OpenAIClient(settings.OPENAI_API_KEY)
    # For demo, we simulate streaming
    async def mock_stream():
        words = ["Hello", " there!", " How", " can", " I", " help", " you?"]
        for word in words:
            yield f"data: {word}\n\n"
            await asyncio.sleep(0.1)  # simulate token generation delay
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        mock_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",   # disable nginx buffering
        }
    )
```

---

## 7. ⚡ aiohttp — The Alternative

Python

```
"""
aiohttp: older async HTTP library.
Still widely used, especially in data pipelines and scrapers.
Some teams prefer it for its mature ecosystem.
"""
import aiohttp
import asyncio
from typing import List, Dict, Optional


async def aiohttp_basics():
    """aiohttp fundamentals."""
    print("\n=== aiohttp Basics ===\n")

    # Session = like httpx.AsyncClient
    async with aiohttp.ClientSession(
        headers={"User-Agent": "MyApp/1.0"},
        timeout=aiohttp.ClientTimeout(total=30, connect=5),
        connector=aiohttp.TCPConnector(
            limit=100,           # max concurrent connections
            limit_per_host=10,   # max per host
            enable_cleanup_closed=True,
        )
    ) as session:

        # GET
        async with session.get("https://httpbin.org/get") as response:
            print(f"Status: {response.status}")
            print(f"Content-Type: {response.content_type}")
            data = await response.json()
            print(f"URL: {data['url']}")

        # POST with JSON
        async with session.post(
            "https://httpbin.org/post",
            json={"name": "Alice", "role": "admin"}
        ) as response:
            data = await response.json()
            print(f"Posted: {data['json']}")

        # GET with params
        async with session.get(
            "https://httpbin.org/get",
            params={"page": 1, "limit": 20}
        ) as response:
            data = await response.json()
            print(f"URL: {data['url']}")

        # Stream large download
        async with session.get("https://httpbin.org/bytes/10000") as response:
            total = 0
            async for chunk, _ in response.content.iter_chunks():
                total += len(chunk)
            print(f"Downloaded: {total} bytes")


# ─────────────────────────────────────────
# HIGH-PERFORMANCE SCRAPER with aiohttp
# ─────────────────────────────────────────
async def async_scraper(
    urls: List[str],
    max_concurrent: int = 10,
    delay: float = 0.1,
) -> List[Dict]:
    """
    Scrape multiple URLs concurrently.
    Respects rate limits with semaphore.
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []

    connector = aiohttp.TCPConnector(
        limit=max_concurrent,
        force_close=False,
        enable_cleanup_closed=True,
    )

    timeout = aiohttp.ClientTimeout(total=30, connect=5)

    async with aiohttp.ClientSession(
        connector=connector,
        timeout=timeout,
        headers={
            "User-Agent": "MyBot/1.0 (Research)",
            "Accept": "text/html,application/json"
        }
    ) as session:

        async def fetch_url(url: str, index: int) -> Dict:
            async with semaphore:
                start = asyncio.get_event_loop().time()
                try:
                    await asyncio.sleep(delay)  # rate limiting

                    async with session.get(url) as response:
                        content = await response.text()
                        elapsed = asyncio.get_event_loop().time() - start

                        return {
                            "url": url,
                            "index": index,
                            "status": response.status,
                            "size": len(content),
                            "elapsed_ms": elapsed * 1000,
                            "success": True
                        }

                except aiohttp.ClientError as e:
                    return {
                        "url": url,
                        "index": index,
                        "status": 0,
                        "error": str(e),
                        "success": False
                    }
                except asyncio.TimeoutError:
                    return {
                        "url": url,
                        "index": index,
                        "status": 0,
                        "error": "timeout",
                        "success": False
                    }

        tasks = [fetch_url(url, i) for i, url in enumerate(urls)]
        results = await asyncio.gather(*tasks)

    return results


# Test scraper
async def run_scraper():
    urls = [
        "https://httpbin.org/get",
        "https://httpbin.org/ip",
        "https://httpbin.org/user-agent",
        "https://httpbin.org/headers",
        "https://httpbin.org/status/200",
    ]

    import time
    start = time.perf_counter()
    results = await async_scraper(urls, max_concurrent=5)
    elapsed = time.perf_counter() - start

    successful = sum(1 for r in results if r["success"])
    print(f"\nScraper results: {successful}/{len(results)} successful")
    print(f"Total time: {elapsed:.2f}s")
    for r in results:
        status = "✓" if r["success"] else "✗"
        print(f"  {status} {r['url']}: {r.get('status')} ({r.get('elapsed_ms', 0):.0f}ms)")


asyncio.run(aiohttp_basics())
asyncio.run(run_scraper())
```

---

## 8. 🧪 Testing HTTP Clients

Python

```
"""
Testing async HTTP clients without making real HTTP calls.
Use httpx's built-in mock transport or pytest-httpx.
"""
import asyncio
import pytest
import httpx
from unittest.mock import AsyncMock, patch
from typing import Dict


# ─────────────────────────────────────────
# HTTPX MOCK TRANSPORT
# ─────────────────────────────────────────
class MockTransport(httpx.AsyncBaseTransport):
    """
    In-memory HTTP transport for testing.
    No actual HTTP calls made.
    """

    def __init__(self, responses: Dict[str, httpx.Response]):
        """
        responses: {url: response} mapping
        """
        self.responses = responses
        self.requests = []   # track all requests made

    async def handle_async_request(
        self,
        request: httpx.Request
    ) -> httpx.Response:
        """Called for every request — return mock response."""
        self.requests.append(request)
        url = str(request.url)

        # Find matching response
        for pattern, response in self.responses.items():
            if pattern in url or url == pattern:
                return response

        # Default: 404 if no match
        return httpx.Response(
            404,
            json={"error": f"No mock for {url}"}
        )


def make_response(
    status: int = 200,
    json_data: Dict = None,
    text: str = None,
    headers: Dict = None,
) -> httpx.Response:
    """Helper to create mock responses."""
    content = b""
    response_headers = headers or {}

    if json_data is not None:
        import json
        content = json.dumps(json_data).encode()
        response_headers["content-type"] = "application/json"
    elif text:
        content = text.encode()
        response_headers["content-type"] = "text/plain"

    return httpx.Response(
        status_code=status,
        content=content,
        headers=response_headers,
    )


# ─────────────────────────────────────────
# EXAMPLE: Testing GitHubClient
# ─────────────────────────────────────────
async def test_github_client_with_mock():
    """Test GitHub client without real API calls."""

    # Define mock responses
    mock_transport = MockTransport({
        "https://api.github.com/users/testuser": make_response(
            json_data={
                "login": "testuser",
                "name": "Test User",
                "email": "test@github.com",
                "followers": 1234,
                "public_repos": 42,
            }
        ),
        "https://api.github.com/users/testuser/repos": make_response(
            json_data=[
                {"id": 1, "name": "repo-one", "stargazers_count": 100},
                {"id": 2, "name": "repo-two", "stargazers_count": 50},
            ]
        ),
        "https://api.github.com/repos/testuser/repo-one": make_response(
            json_data={"id": 1, "name": "repo-one", "stargazers_count": 100}
        ),
        "https://api.github.com/repos/testuser/repo-one/languages": make_response(
            json_data={"Python": 15000, "JavaScript": 3000}
        ),
    })

    # Create client with mock transport
    async with httpx.AsyncClient(
        base_url="https://api.github.com",
        transport=mock_transport,
    ) as mock_client:
        # Test user profile fetch
        response = await mock_client.get("/users/testuser")
        user = response.json()

        assert user["login"] == "testuser"
        assert user["followers"] == 1234
        assert user["public_repos"] == 42
        print(f"✓ User profile: {user['name']}")

        # Test repos fetch
        response = await mock_client.get("/users/testuser/repos")
        repos = response.json()

        assert len(repos) == 2
        assert repos[0]["name"] == "repo-one"
        print(f"✓ User repos: {[r['name'] for r in repos]}")

        # Test languages fetch
        response = await mock_client.get("/repos/testuser/repo-one/languages")
        languages = response.json()

        assert "Python" in languages
        print(f"✓ Languages: {list(languages.keys())}")

        # Verify requests made
        print(f"✓ Total requests: {len(mock_transport.requests)}")


# ─────────────────────────────────────────
# USING pytest-httpx (cleaner test library)
# ─────────────────────────────────────────
# pip install pytest-httpx pytest-asyncio

# conftest.py
# from pytest_httpx import HTTPXMock

# tests/test_clients.py
# @pytest.mark.asyncio
# async def test_fetch_user(httpx_mock):
#     httpx_mock.add_response(
#         url="https://api.github.com/users/alice",
#         json={"login": "alice", "name": "Alice"}
#     )
#
#     async with GitHubClient() as client:
#         user = await client.get_user("alice")
#         assert user["login"] == "alice"


# ─────────────────────────────────────────
# INTEGRATION TEST with real (test) server
# ─────────────────────────────────────────
from fastapi import FastAPI
from httpx import AsyncClient, ASGITransport


async def test_with_fastapi_testclient():
    """Test client against your own FastAPI app."""
    app = FastAPI()

    @app.get("/api/data")
    async def get_data():
        return {"data": "value", "count": 42}

    @app.post("/api/process")
    async def process(payload: dict):
        return {"processed": True, "input": payload}

    # Test against the app directly (no network!)
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        # Test GET
        response = await client.get("/api/data")
        assert response.status_code == 200
        assert response.json()["count"] == 42
        print(f"✓ GET /api/data: {response.json()}")

        # Test POST
        response = await client.post(
            "/api/process",
            json={"key": "value", "number": 123}
        )
        assert response.status_code == 200
        assert response.json()["processed"] is True
        print(f"✓ POST /api/process: {response.json()}")

        # Test concurrent requests
        import asyncio
        responses = await asyncio.gather(*[
            client.get("/api/data") for _ in range(10)
        ])
        assert all(r.status_code == 200 for r in responses)
        print(f"✓ 10 concurrent requests: all 200")


# Run tests
asyncio.run(test_github_client_with_mock())
asyncio.run(test_with_fastapi_testclient())
```

---

## 9. 🎯 Complete Production Example

Python

```
# A complete FastAPI service that calls multiple external APIs
import asyncio
import httpx
from fastapi import FastAPI, Depends, HTTPException, BackgroundTasks
from typing import Dict, Any, List, Optional
from pydantic import BaseModel

app = FastAPI(title="External Services Integration")

# ─────────────────────────────────────────
# SHARED HTTP CLIENT (singleton)
# ─────────────────────────────────────────
http_client: Optional[httpx.AsyncClient] = None


@app.on_event("startup")
async def startup():
    global http_client
    http_client = httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=30.0),
        limits=httpx.Limits(
            max_connections=100,
            max_keepalive_connections=20,
        ),
        headers={"User-Agent": "MyService/1.0"},
        follow_redirects=True,
    )
    print("HTTP client pool initialized")


@app.on_event("shutdown")
async def shutdown():
    if http_client:
        await http_client.aclose()
    print("HTTP client pool closed")


def get_http_client() -> httpx.AsyncClient:
    return http_client


# ─────────────────────────────────────────
# SERVICE CLASSES
# ─────────────────────────────────────────
class UserEnrichmentService:
    """
    Enriches user data from multiple external APIs.
    All calls happen concurrently.
    """

    def __init__(self, client: httpx.AsyncClient):
        self.client = client

    async def get_ip_location(self, ip: str) -> Dict:
        """Get geographic info for an IP address."""
        try:
            response = await self.client.get(
                f"https://ipapi.co/{ip}/json/"
            )
            if response.is_success:
                data = response.json()
                return {
                    "country": data.get("country_name"),
                    "city": data.get("city"),
                    "timezone": data.get("timezone"),
                }
        except Exception:
            pass
        return {}

    async def validate_email_domain(self, email: str) -> Dict:
        """Check if email domain has valid MX records."""
        domain = email.split("@")[-1]
        try:
            response = await self.client.get(
                "https://dns.google/resolve",
                params={"name": domain, "type": "MX"}
            )
            data = response.json()
            has_mx = bool(data.get("Answer"))
            return {"domain": domain, "has_mx_records": has_mx}
        except Exception:
            return {"domain": domain, "has_mx_records": None}

    async def get_github_profile(self, username: str) -> Dict:
        """Fetch GitHub profile if user has one."""
        try:
            response = await self.client.get(
                f"https://api.github.com/users/{username}"
            )
            if response.is_success:
                data = response.json()
                return {
                    "github_url": data.get("html_url"),
                    "public_repos": data.get("public_repos"),
                    "followers": data.get("followers"),
                }
        except Exception:
            pass
        return {}

    async def enrich_user(
        self,
        email: str,
        ip: Optional[str] = None,
        github_username: Optional[str] = None,
    ) -> Dict:
        """
        Enrich user with data from multiple sources.
        All API calls happen CONCURRENTLY.
        """
        tasks = {}

        # Email validation always runs
        tasks["email_info"] = self.validate_email_domain(email)

        # Optional enrichments
        if ip:
            tasks["location"] = self.get_ip_location(ip)
        if github_username:
            tasks["github"] = self.get_github_profile(github_username)

        # All run concurrently
        keys = list(tasks.keys())
        results = await asyncio.gather(
            *tasks.values(),
            return_exceptions=True
        )

        enrichment = {}
        for key, result in zip(keys, results):
            if not isinstance(result, Exception):
                enrichment[key] = result

        return enrichment


# ─────────────────────────────────────────
# API ENDPOINTS
# ─────────────────────────────────────────
class UserRegistration(BaseModel):
    email: str
    name: str
    github_username: Optional[str] = None


@app.post("/users/register")
async def register_user(
    data: UserRegistration,
    background_tasks: BackgroundTasks,
    client: httpx.AsyncClient = Depends(get_http_client),
    request_ip: str = "8.8.8.8",   # would come from request
) -> Dict:
    """
    Register user with concurrent external enrichment.
    """
    service = UserEnrichmentService(client)

    # Enrich user data concurrently while registering
    enrichment = await service.enrich_user(
        email=data.email,
        ip=request_ip,
        github_username=data.github_username,
    )

    # In production: save to database
    user = {
        "id": 42,
        "name": data.name,
        "email": data.email,
        "enrichment": enrichment,
    }

    # Send welcome email in background (doesn't block response)
    async def send_welcome():
        await asyncio.sleep(0.1)   # simulate email sending
        print(f"Welcome email sent to {data.email}")

    background_tasks.add_task(send_welcome)

    return {
        "user": user,
        "message": "Registration successful"
    }


@app.get("/external/status")
async def check_external_services(
    client: httpx.AsyncClient = Depends(get_http_client),
) -> Dict:
    """
    Check health of all external services concurrently.
    """
    async def check_service(name: str, url: str) -> Dict:
        try:
            start = asyncio.get_event_loop().time()
            response = await client.get(url)
            elapsed = (asyncio.get_event_loop().time() - start) * 1000
            return {
                "name": name,
                "status": "up" if response.is_success else "degraded",
                "latency_ms": round(elapsed, 1),
            }
        except Exception as e:
            return {"name": name, "status": "down", "error": str(e)}

    services = [
        ("GitHub API", "https://api.github.com"),
        ("IP Info", "https://ipapi.co/8.8.8.8/json/"),
        ("DNS", "https://dns.google/resolve?name=example.com&type=A"),
    ]

    start = asyncio.get_event_loop().time()
    results = await asyncio.gather(*[
        check_service(name, url) for name, url in services
    ])
    total_ms = (asyncio.get_event_loop().time() - start) * 1000

    return {
        "services": results,
        "total_check_time_ms": round(total_ms, 1),
        "all_up": all(r["status"] == "up" for r in results),
    }
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│                    ASYNC HTTP CLIENTS                          │
│                                                                │
│  HTTPX (recommended):                                          │
│  httpx.AsyncClient() → async HTTP client                      │
│  base_url, timeout, limits, headers, auth                     │
│  .get() .post() .put() .patch() .delete()                     │
│  .stream() → for large responses                              │
│  response.json() .text .content                               │
│  response.raise_for_status() → raise on 4xx/5xx              │
│                                                                │
│  PRODUCTION PATTERN:                                           │
│  One client per app (not per request!)                        │
│  Create at startup, close at shutdown                         │
│  Connection pool shared across all requests                   │
│                                                                │
│  RETRY PATTERN:                                                │
│  Retry: ConnectError, TimeoutError, 5xx, 429                 │
│  Don't retry: 4xx (client's fault)                           │
│  Exponential backoff: 1s, 2s, 4s, 8s...                      │
│  Add jitter to prevent thundering herd                        │
│                                                                │
│  CIRCUIT BREAKER:                                              │
│  CLOSED → failures → OPEN → timeout → HALF_OPEN → CLOSED     │
│  Prevents cascading failures from flaky external services     │
│                                                                │
│  CONCURRENT REQUESTS:                                          │
│  await asyncio.gather(req1, req2, req3)  ← all at once       │
│  Semaphore(N) → limit concurrent requests                    │
│  return_exceptions=True → don't fail on partial failure       │
│                                                                │
│  WEBHOOKS:                                                     │
│  Sending: HMAC signature for verification                     │
│  Receiving: verify signature before processing                │
│  Always return 200 fast, process async                        │
│                                                                │
│  TESTING:                                                      │
│  MockTransport → no real HTTP calls                          │
│  ASGITransport → test your own FastAPI app                   │
│  pytest-httpx → clean test fixtures                          │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  Why should you never create a new httpx.AsyncClient
    per request in FastAPI?
2.  What are the four timeout types in httpx.Timeout?
    What does each control?
3.  What HTTP errors should you retry? Which should you not?
4.  What is exponential backoff and why add jitter?
5.  What is the Circuit Breaker pattern?
    What are its three states?
6.  What is the difference between httpx and aiohttp?
    When might you choose aiohttp?
7.  How do you handle streaming large responses?
8.  What is a webhook signing secret and why use it?
9.  How do you prevent replay attacks in webhook verification?
10. How do you test httpx clients without making real HTTP calls?
11. What does return_exceptions=True do in asyncio.gather?
12. What is ASGITransport used for in testing?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Weather Service Client
# Build a WeatherClient that wraps open-meteo.com (free, no key needed):
# URL: https://api.open-meteo.com/v1/forecast
# async def get_current_weather(lat, lon) -> dict
# async def get_forecast(lat, lon, days=7) -> list[dict]
# async def get_weather_for_cities(cities: list[tuple]) -> list[dict]
#   (cities as (lat, lon) tuples, all fetched concurrently)
# Handle errors gracefully, add retry logic

# Exercise 2: Rate-Limited API Client
# Build a client for any free API (e.g., JSONPlaceholder)
# That enforces:
# - Max 10 requests per second
# - Max 3 concurrent requests
# - Auto-retry on 429 with Retry-After header
# - Circuit breaker after 5 consecutive failures
# Benchmark: how many requests can you make per second?

# Exercise 3: Webhook System
# Build a complete webhook system:
# POST /webhooks/register   → register a webhook URL for events
# GET  /webhooks            → list registered webhooks
# DELETE /webhooks/{id}     → unregister
# POST /events/trigger      → trigger an event, sends to all registered webhooks
# Requirements:
# - HMAC signing for security
# - Async delivery with retries
# - Track delivery status per webhook
# - Webhook health monitoring (disable failing ones)

# Exercise 4: API Aggregator
# Build an endpoint that aggregates data from multiple sources:
# GET /user/{github_username}/profile
# Returns:
# - GitHub profile (github API)
# - Their most starred repo details
# - Programming language breakdown
# - Top 3 contributors of their most starred repo
# All fetched concurrently, with graceful degradation
# (if one fails, still return what succeeded)

# Exercise 5: HTTP Client with Full Observability
# Wrap httpx.AsyncClient to add:
# - Structured logging for every request/response
# - Prometheus metrics:
#     http_requests_total{method, url, status}
#     http_request_duration_seconds{method, url}
# - Distributed tracing headers (X-Request-ID propagation)
# - Automatic retry with logged attempts
# - Circuit breaker per host
# Use it in a FastAPI app and verify metrics are collected
```

---

## 🎉 Phase 5 Complete!

text

```
✅ 5.1 — Asyncio Fundamentals
         event loop, coroutines, gather, tasks, locks, queues

✅ 5.2 — Async Database Operations
         asyncpg, SQLAlchemy async, repositories, FastAPI deps

✅ 5.3 — Threading vs Multiprocessing vs Async
         GIL, when to use each, run_in_executor, decision guide

✅ 5.4 — Async HTTP Clients
         httpx, aiohttp, retry, circuit breaker,
         webhooks, testing, real API clients
```

**You can now build fully async, non-blocking backends  
that handle thousands of concurrent operations efficiently.**