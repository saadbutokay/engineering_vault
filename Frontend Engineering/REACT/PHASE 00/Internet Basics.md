## Concept Explanation

### Client-Server Model

Every interaction on the web follows a request-response pattern between two parties:

- **Client:** The device/application requesting information (your browser, mobile app, or any frontend)
- **Server:** The machine that stores and provides the requested information (where websites, APIs, and databases live)

When you type a URL in your browser, you are the client asking a server: "Give me this webpage." The server processes the request and sends back a response (usually HTML, CSS, JavaScript, or data in JSON format).

This is identical to backend work you already know. In Python, when you built an API with Flask or Django, your code was the server. Now, as a frontend engineer, you are primarily building the client that talks to that server.

### HTTP and HTTPS

HTTP (HyperText Transfer Protocol) is the language/protocol that clients and servers use to communicate. It defines rules such as:

- How a request should be structured
- How a response should be structured
- What methods exist (GET, POST, PUT, DELETE, PATCH)
- What status codes mean (200 OK, 404 Not Found, 500 Server Error, etc.)

HTTPS is HTTP with an added security layer called TLS/SSL encryption. It ensures that data traveling between client and server cannot be read or tampered with by anyone intercepting it. Every production website today must use HTTPS. Browsers actively warn users and penalize SEO rankings for non-HTTPS sites.

Key HTTP concepts you need to know as a frontend engineer:

| Concept | Purpose |
| --- | --- |
| Request Method | Defines action (GET = fetch data, POST = send data, etc.) |
| Headers | Metadata about the request/response (content type, authorization tokens, etc.) |
| Body | The actual data being sent (used in POST, PUT, PATCH) |
| Status Code | Tells you the result of the request (success, failure, redirect) |
| URL | The address identifying the resource being requested |

Common status codes you will deal with constantly:

- 200 — OK, request succeeded
- 201 — Created, resource successfully created
- 301/302 — Redirect
- 400 — Bad Request (client sent invalid data)
- 401 — Unauthorized (missing/invalid authentication)
- 403 — Forbidden (authenticated but not allowed)
- 404 — Not Found
- 500 — Internal Server Error

### DNS (Domain Name System)

Computers communicate using IP addresses (e.g., 142.250.182.14), not human-readable names like google.com. DNS is essentially the internet's phonebook. It translates domain names into IP addresses.

The process when you type a URL:

1. Browser checks its own cache for the IP address
2. If not cached, it asks a DNS resolver (usually provided by your ISP)
3. The resolver queries root servers, then top-level domain (TLD) servers, then authoritative servers
4. The IP address is returned to your browser
5. Browser connects to that IP address and initiates the HTTP/HTTPS request

This entire process typically takes milliseconds but is foundational to understanding why websites sometimes load slowly on first visit (DNS lookup time) versus subsequent visits (cached).

## Basic Syntax Example

A minimal example showing the client making an HTTP request to a server using JavaScript's built-in fetch function. This demonstrates the client-server relationship directly in code.

```js
// Basic GET request - client asking server for data
fetch('https://api.example.com/users')
  .then(response => response.json())
  .then(data => console.log(data));
```

```js
// Basic POST request - client sending data to server
fetch('https://api.example.com/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'John Doe', email: 'john@example.com' })
})
  .then(response => response.json())
  .then(data => console.log(data));
```

## Industry-Level Code Snippet

In real production applications, requests are wrapped with proper error handling, timeout handling, status code checks, and often organized into reusable service functions rather than scattered fetch calls.

```js
// api/client.js
// A production-grade HTTP client wrapper used across a real application

const BASE_URL = 'https://api.production-app.com';

async function apiRequest(endpoint, options = {}) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 8000); // 8s timeout

  try {
    const response = await fetch(`${BASE_URL}${endpoint}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getAuthToken()}`,
        ...options.headers
      },
      signal: controller.signal
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      // Handle different HTTP status codes explicitly
      if (response.status === 401) {
        throw new Error('Unauthorized: Please log in again');
      }
      if (response.status === 404) {
        throw new Error('Resource not found');
      }
      if (response.status >= 500) {
        throw new Error('Server error, please try again later');
      }
      throw new Error(`Request failed with status ${response.status}`);
    }

    return await response.json();

  } catch (error) {
    clearTimeout(timeoutId);

    if (error.name === 'AbortError') {
      throw new Error('Request timed out');
    }
    throw error;

  }
}

function getAuthToken() {
  return localStorage.getItem('authToken') || '';
}

// Usage elsewhere in the application
export async function getUser(userId) {
  return apiRequest(`/users/${userId}`, { method: 'GET' });
}

export async function createUser(userData) {
  return apiRequest('/users', {
    method: 'POST',
    body: JSON.stringify(userData)
  });
}
```

This pattern — a centralized API client with timeout control, authentication headers, and explicit status code handling — is standard in real-world frontend codebases, whether using plain JavaScript, React, or Next.js.
