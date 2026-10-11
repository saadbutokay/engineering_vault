Examples target **.NET 10 (LTS)** and **C# 14**. The API example uses ASP.NET Core's built-in exception-handler and Problem Details services.

## 1. Conceptual Foundation

### 1.1 What Is Exception Handling?

**Exception handling** is C#'s structured mechanism for signaling, propagating, and responding to failures that interrupt normal execution. Examples include invalid method arguments, unavailable resources, violated business rules, and network failures.

When code throws an object derived from `System.Exception`, the runtime searches outward through the call stack for a compatible `catch` clause. If no matching handler is found, the exception escapes to the host. In a console application an unhandled exception normally terminates the program; in an ASP.NET Core application, configured exception-handling middleware can catch request exceptions and produce a controlled response. Don't assume a web API has that middleware unless the application configures it.

A simplified request path might look like this:

```text
API exception handler
        ↑ handles or maps the failure
Order service
        ↑ exception propagates
Inventory service  ── throws InsufficientStockException
```

A catch block should handle an exception only when it can recover, add useful context, perform required rollback/cleanup, or translate the failure at an appropriate boundary. Otherwise, let it propagate.

## 2. `try`, `catch`, and `finally`

A `try` block contains code that may fail. A `catch` handles a matching exception type. A `finally` block runs when control leaves the associated `try`/`catch` flow, whether normally or because of an exception.

```csharp
try
{
    // Code that might throw.
}
catch (IOException)
{
    // Recover here only if the application can take a useful action.
    // Otherwise don't swallow the exception.
    throw;
}
finally
{
    // Release resources or restore state that isn't handled by `using`.
}
```

### 2.1 Catch-Block Ordering

Catch clauses are examined from top to bottom, and at most one matching `catch` runs. Put more-derived, specific exception types before their base types. For example, `catch (IOException)` must come before `catch (Exception)`; otherwise the later clause is unreachable and the compiler reports an error. A general `catch` should be used sparingly and placed last.

### 2.2 Exception Filters (`when`)

A `when` filter adds a Boolean condition to a catch clause. The filter is evaluated while the runtime searches for a handler, before stack unwinding. If the filter is false, that clause doesn't handle the exception and the search continues.

```csharp
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public static class HttpRequestDemo
{
    public static async Task<HttpResponseMessage> SendAsync(
        HttpClient client,
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        try
        {
            return await client.SendAsync(request, cancellationToken);
        }
        catch (HttpRequestException ex)
            when (ex.StatusCode == HttpStatusCode.TooManyRequests)
        {
            // Apply a specific rate-limit policy here; this example propagates it.
            throw;
        }
    }
}
```

Filters are useful when the exception type is the same but the recovery policy depends on additional runtime state. Keep filter expressions simple and side-effect-free.

### 2.3 Rethrowing Without Losing the Stack Trace

Inside a `catch`, `throw;` rethrows the current exception and preserves its original stack trace. `throw ex;` resets the stack trace to the rethrow location, obscuring where the failure began.

```csharp
catch (Exception ex)
{
    Console.Error.WriteLine(ex); // Console example; use ILogger in an application.
    throw; // Preserves the original stack trace.
}
```

If you intentionally wrap an exception in a more specific one, pass the original as the **inner exception**: `throw new CatalogUnavailableException("Catalog request failed.", ex);`.

### 2.4 Cleanup and `finally`

Use `using` or `await using` for resources that implement `IDisposable` or `IAsyncDisposable`; disposal occurs even if an exception is thrown. Use `finally` for cleanup or state restoration that isn't naturally expressed by a `using` scope. A `finally` block normally runs on return, rethrow, and ordinary exception propagation, but no managed cleanup is guaranteed if the process is forcibly terminated (for example, `Environment.FailFast` or an OS shutdown). Avoid throwing a new exception from `finally`, because it can hide an exception already in flight.

## 3. Common Exception Types

| Exception type | Typical meaning |
|---|---|
| `ArgumentException` | A supplied argument is invalid. |
| `ArgumentNullException` | A required argument is `null`. |
| `ArgumentOutOfRangeException` | An argument is outside the method's accepted range. |
| `InvalidOperationException` | An operation isn't valid for the object's current state. |
| `KeyNotFoundException` | A dictionary lookup requested a missing key. |
| `IndexOutOfRangeException` | An array index is outside its valid bounds; usually a programming error to fix rather than catch. |
| `NullReferenceException` | Code dereferenced `null`; normally prevent it with correct state and nullable analysis rather than catching it. |
| `TimeoutException` | An operation exceeded a time limit. |
| `OperationCanceledException` | An operation observed a cancellation request. `TaskCanceledException` derives from it. |

For asynchronous code, `OperationCanceledException` is the general cancellation type to expect; don't assume every canceled operation throws `TaskCanceledException`. Cancellation is often part of normal control flow. Handle it only to clean up or translate a deliberate cancellation policy, and otherwise let it propagate. Exceptions thrown during an `async` method are generally stored in its returned `Task` and observed when the task is awaited.

Many built-in exceptions derive from `SystemException`. Custom exceptions should derive from `Exception` or an application-specific exception base—not from `SystemException` or the legacy `ApplicationException` convention.

## 4. Custom Exceptions

Create a custom exception when callers need to distinguish a meaningful failure category or inspect useful structured data. Use names ending in `Exception`, choose a clear message, and add properties only when callers have a programmatic use for them.

For general-purpose public exception types, current .NET guidance recommends the three common constructors: parameterless, message-only, and message plus inner exception. This is constructor/API guidance; don't add legacy serialization boilerplate solely on the assumption that it is required for ordinary modern .NET exception handling. For a domain exception that must always carry specific data, expose the constructors that preserve that valid state rather than adding meaningless overloads.

```csharp
using System;

namespace MyBackendApp.Core.Exceptions;

public sealed class CatalogUnavailableException : Exception
{
    public CatalogUnavailableException()
    {
    }

    public CatalogUnavailableException(string message)
        : base(message)
    {
    }

    public CatalogUnavailableException(string message, Exception innerException)
        : base(message, innerException)
    {
    }
}
```

A data-rich business exception can expose structured properties in addition to its message. See `DomainExceptions.cs` in the backend example for `InsufficientStockException` and related types.

## 5. Performance and Design Guidance

- **Use exceptions for exceptional failures, not routine decisions.** Throwing is substantially more expensive than a normal return path because the runtime must create and propagate an exception and capture diagnostic state. For expected cases in a hot or common path, prefer checks, `TryParse`, `TryGetValue`, a Boolean, or an explicit `Result<T>` pattern.
- **Don't silently swallow exceptions.** An empty catch hides a failure and may leave the application in an invalid state.
- **Catch only what you can handle.** If recovery isn't possible at the current layer, let the exception reach a layer that can act—often the process or request boundary.
- **Preserve state when an operation fails.** If a multi-step operation partly changes state before throwing, roll back or otherwise restore a valid state before rethrowing.
- **Clean up deterministically.** Prefer `using`/`await using` for disposable resources; use `finally` for other cleanup and rollback.
- **Avoid duplicate logging.** Log at the layer that owns the handling decision instead of logging and rethrowing at every layer.

## 6. Basic Syntax Example

```csharp
using System;

namespace MyBackendApp.Basics;

public sealed class InsufficientFundsException : Exception
{
    public decimal AttemptedAmount { get; }
    public decimal AvailableBalance { get; }

    public InsufficientFundsException(decimal attemptedAmount, decimal availableBalance)
        : base($"Cannot withdraw {attemptedAmount:C}; only {availableBalance:C} is available.")
    {
        AttemptedAmount = attemptedAmount;
        AvailableBalance = availableBalance;
    }
}

public sealed class BankAccount
{
    public decimal Balance { get; private set; } = 100.00m;

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
        {
            throw new ArgumentOutOfRangeException(
                nameof(amount), "Withdrawal amount must be positive.");
        }

        if (amount > Balance)
        {
            throw new InsufficientFundsException(amount, Balance);
        }

        Balance -= amount;
    }
}

public static class ExceptionHandlingDemo
{
    public static void Run()
    {
        var account = new BankAccount();

        try
        {
            account.Withdraw(500.00m);
        }
        catch (InsufficientFundsException ex)
        {
            Console.WriteLine($"Transaction declined: {ex.Message}");
        }
        catch (ArgumentOutOfRangeException ex)
        {
            Console.WriteLine($"Invalid input: {ex.Message}");
        }
        finally
        {
            Console.WriteLine($"Current balance remains: {account.Balance:C}");
        }
    }
}
```

The `finally` block runs after the matching catch and shows that the balance remains unchanged after the rejected withdrawal. The service throws a specific exception; the caller decides how to respond.

## 7. Backend Example: Domain Exceptions and ASP.NET Core Handling

For an API, application code can throw a meaningful exception when that is the chosen domain design, while a single boundary handler maps known failures to appropriate HTTP statuses. Routine request validation can instead return validation errors without throwing. Never send raw stack traces or unexpected exception messages to clients.

ASP.NET Core provides `IExceptionHandler` and `IProblemDetailsService`; a custom handler doesn't need to redefine `HttpContext`, `RequestDelegate`, or response types. ASP.NET Core documentation commonly describes its `ProblemDetails` responses as RFC 7807-style. **RFC 9457 is the current IETF Problem Details specification and obsoletes RFC 7807.**

### 7.1 Domain Exceptions — `DomainExceptions.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Exceptions;

public abstract class DomainException : Exception
{
    protected DomainException(string message)
        : base(message)
    {
    }

    protected DomainException(string message, Exception innerException)
        : base(message, innerException)
    {
    }
}

public sealed class InsufficientStockException : DomainException
{
    public string Sku { get; }
    public int RequestedQuantity { get; }
    public int AvailableQuantity { get; }

    public InsufficientStockException(string sku, int requestedQuantity, int availableQuantity)
        : base($"Insufficient stock for SKU '{sku}': requested {requestedQuantity}, available {availableQuantity}.")
    {
        Sku = sku;
        RequestedQuantity = requestedQuantity;
        AvailableQuantity = availableQuantity;
    }
}

public sealed class SkuNotFoundException : DomainException
{
    public string Sku { get; }

    public SkuNotFoundException(string sku)
        : base($"SKU '{sku}' was not found.")
    {
        Sku = sku;
    }
}

public sealed class OrderNotFoundException : DomainException
{
    public Guid OrderId { get; }

    public OrderNotFoundException(Guid orderId)
        : base($"Order '{orderId}' was not found.")
    {
        OrderId = orderId;
    }
}
```

The custom properties are useful to application code and logs. The API handler below deliberately returns safe, stable client details rather than echoing these exception messages.

### 7.2 Application Service — `OrderFulfillmentService.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using MyBackendApp.Core.Exceptions;

namespace MyBackendApp.Core.Services;

public sealed class OrderFulfillmentService
{
    private readonly Dictionary<string, int> _stockLevels;

    public OrderFulfillmentService(Dictionary<string, int> stockLevels)
    {
        ArgumentNullException.ThrowIfNull(stockLevels);
        _stockLevels = stockLevels;
    }

    public void ReserveStock(string sku, int quantity)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(sku);
        if (quantity <= 0)
        {
            throw new ArgumentOutOfRangeException(
                nameof(quantity), "Quantity must be positive.");
        }

        if (!_stockLevels.TryGetValue(sku, out int available))
        {
            throw new SkuNotFoundException(sku);
        }

        if (available < quantity)
        {
            throw new InsufficientStockException(sku, quantity, available);
        }

        _stockLevels[sku] = available - quantity;
    }
}
```

The service uses `TryGetValue` for an ordinary missing-key check and throws only when it needs to communicate a domain failure. It doesn't catch its own domain exception; the API boundary decides how to represent it. The dictionary is only a compact example: a production inventory service needs an atomic database update or transaction so concurrent reservations can't oversell stock.

### 7.3 Central API Handler — `ApiExceptionHandler.cs`

```csharp
#nullable enable
using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using MyBackendApp.Core.Exceptions;

namespace MyBackendApp.Api.Middleware;

public sealed class ApiExceptionHandler : IExceptionHandler
{
    private readonly ILogger<ApiExceptionHandler> _logger;
    private readonly IProblemDetailsService _problemDetailsService;

    public ApiExceptionHandler(
        ILogger<ApiExceptionHandler> logger,
        IProblemDetailsService problemDetailsService)
    {
        _logger = logger;
        _problemDetailsService = problemDetailsService;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (httpContext.Response.HasStarted)
        {
            return false;
        }

        // Don't turn an aborted request into a normal 500 response.
        if (exception is OperationCanceledException && cancellationToken.IsCancellationRequested)
        {
            return false;
        }

        ProblemDetails problem = exception switch
        {
            OrderNotFoundException => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Order not found",
                Type = "https://api.example.com/problems/order-not-found",
                Detail = "The requested order was not found."
            },
            SkuNotFoundException => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Product not found",
                Type = "https://api.example.com/problems/product-not-found",
                Detail = "The requested product was not found."
            },
            InsufficientStockException => new ProblemDetails
            {
                Status = StatusCodes.Status409Conflict,
                Title = "Insufficient stock",
                Type = "https://api.example.com/problems/insufficient-stock",
                Detail = "The requested quantity is not currently available."
            },
            ArgumentException => new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Invalid request",
                Type = "https://api.example.com/problems/invalid-request",
                Detail = "One or more request values are invalid."
            },
            DomainException => new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Business rule violation",
                Type = "https://api.example.com/problems/business-rule-violation",
                Detail = "The request cannot be completed."
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "An unexpected error occurred",
                Type = "https://api.example.com/problems/internal-error",
                Detail = "Please try again later."
            }
        };

        int statusCode = problem.Status ?? StatusCodes.Status500InternalServerError;
        httpContext.Response.StatusCode = statusCode;
        problem.Instance = httpContext.Request.Path.ToString();
        problem.Extensions["traceId"] = httpContext.TraceIdentifier;

        if (exception is DomainException)
        {
            _logger.LogInformation(
                "Mapped {ExceptionType} to HTTP {StatusCode}. TraceId: {TraceId}",
                exception.GetType().Name,
                statusCode,
                httpContext.TraceIdentifier);
        }
        else
        {
            _logger.LogError(
                exception,
                "Unhandled exception mapped to HTTP {StatusCode}. TraceId: {TraceId}",
                statusCode,
                httpContext.TraceIdentifier);
        }

        var problemDetailsContext = new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails = problem
        };

        // If no registered writer accepts the request's Accept header, the status is still set.
        _ = await _problemDetailsService.TryWriteAsync(problemDetailsContext);
        return true;
    }
}
```

Status mappings are API policy, not universal rules. The handler returns `404` for not-found cases, `409` for a stock conflict, `400` for argument validation, and a generic `500` for an unexpected failure. Map `ArgumentException` to `400` only when it represents invalid client input; an internal programming error shouldn't be disguised as a client mistake. The handler never sends an unexpected exception's message or stack trace to the client.

### 7.4 Register the Handler — `Program.cs`

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using MyBackendApp.Api.Middleware;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ApiExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler(new ExceptionHandlerOptions
{
    // The custom handler intentionally maps a missing resource to HTTP 404.
    AllowStatusCode404Response = true
});

// Add the rest of the middleware and endpoints after exception handling.
app.Run();
```

The registered handler is a singleton, so it shouldn't depend on request-scoped services. In .NET 10, ASP.NET Core suppresses its default exception diagnostics when an `IExceptionHandler` returns `true`; this example logs handled domain failures and unexpected exceptions itself. Configure `SuppressDiagnosticsCallback` if the application also wants the middleware's built-in diagnostics and metrics.

## 8. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Exception** | An object derived from `System.Exception` that represents a failure. | Carries failure type, message, inner exception, and diagnostic stack trace. |
| **Stack unwinding** | Runtime traversal from the throw site toward a matching handler. | Explains propagation through layers and when `finally` runs. |
| **Exception filter** | A `when` Boolean condition that controls whether a catch clause matches. | Supports precise handling while the original stack is intact. |
| **`throw;`** | Rethrows the currently caught exception. | Preserves its original stack trace. |
| **Inner exception** | The original exception attached when a higher-level exception wraps it. | Preserves the lower-level cause while adding context. |
| **Custom exception** | An application-defined type derived from `Exception` or a custom base. | Communicates a failure category and any structured data callers need. |
| **Cancellation** | A cooperative request to stop work, typically represented by `OperationCanceledException`. | Usually should propagate as cancellation rather than become a generic server error. |
| **`IExceptionHandler`** | ASP.NET Core's centralized exception-handler interface. | Maps request exceptions at a boundary without catch blocks in every layer. |
| **Problem Details** | A structured HTTP error representation, commonly serialized as `application/problem+json`. | Gives API clients a consistent status, title, detail, and problem type. |

## 9. Official References

- [Exception Handling — C# programming guide](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/exception-handling) — `try`, `catch`, `finally`, filters, and handler ordering.
- [Exception-handling statements — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/exception-handling-statements) — `throw;`, stack traces, filters, and rethrow behavior.
- [Best practices for exceptions — .NET](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions) — recovery, cleanup, cancellation, constructors, and exception design.
- [How to create user-defined exceptions — .NET](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/how-to-create-user-defined-exceptions) — custom exception types and common constructors.
- [Handle errors in ASP.NET Core APIs — .NET 10](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling-api?view=aspnetcore-10.0) — exception middleware and Problem Details responses.
- [`IExceptionHandler` API — ASP.NET Core 10](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.diagnostics.iexceptionhandler?view=aspnetcore-10.0) — centralized request exception handler contract.
- [Exception diagnostics suppressed for handled exceptions — ASP.NET Core 10](https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/exception-handler-diagnostics-suppressed?view=aspnetcore-10.0) — .NET 10 telemetry and logging behavior for `IExceptionHandler`.
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html) — current IETF specification, which obsoletes RFC 7807.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — support tracks; .NET 10 is LTS.
