Examples target **.NET 10 (LTS)** with nullable reference analysis enabled and **C# 14**. Delegates are managed, type-safe objects for invoking methods through a compatible signature; they aren't raw unmanaged function pointers.

## 1. Conceptual Foundation

### 1.1 What Is a Delegate?

A **delegate** is a type-safe object that refers to one or more methods with compatible parameter and return types. It lets code treat behavior as a value: store it in a variable, pass it to another method, return it, and invoke it indirectly.

A delegate is somewhat like a function pointer in C/C++, but it participates in the .NET type system and can be multicast. It can refer to a static method, or to an instance method together with its target object. Invoking a delegate calls the method or methods in its invocation list. For example, `PricingRule rule = ApplyTenPercentDiscount;` is valid when the declared delegate and method signatures match.

### 1.2 Why Delegates Matter in Backend Engineering

Delegates are commonly used for:

- **Callbacks:** pass a small piece of behavior into a reusable operation.
- **LINQ to Objects:** operators such as `Where` and `Select` accept `Func<...>` delegates. (LINQ providers such as `IQueryable<T>` may instead accept expression trees.)
- **Events:** the .NET event pattern is built on delegates, with `event` adding controlled subscription semantics.
- **Small strategies:** pass a predicate, comparer, retry classifier, or transformation without creating a separate interface implementation.
- **Pipelines:** ASP.NET Core's actual `RequestDelegate` is a delegate taking `HttpContext` and returning `Task`; middleware composes request delegates.

Use a named custom delegate when the signature deserves a domain-specific name or doesn't fit `Action`/`Func` (for example, a by-reference signature). For common signatures, built-in delegates are convenient.

## 2. Declaring and Using Delegates

### 2.1 Custom Delegate Types

A custom delegate declaration creates a named type for one exact method signature. A compatible method group can be assigned to it, then invoked directly or through `Invoke`.

```csharp
public delegate decimal PricingRule(decimal basePrice);

public static class CustomDelegateDemo
{
    public static decimal ApplyTenPercentDiscount(decimal basePrice) => basePrice * 0.90m;

    public static decimal Run()
    {
        PricingRule rule = ApplyTenPercentDiscount;
        return rule(100.00m);
    }
}
```

### 2.2 Built-In Generic Delegate Types

The .NET libraries provide common delegate types. `Action` and `Func` support up to 16 input parameters; the last type argument of `Func<...>` is its result type.

| Delegate type | Signature | Typical use |
|---|---|---|
| `Action` | `void Method()` | No parameters, no result. |
| `Action<T1, ..., T16>` | `void Method(T1, ...)` | Parameters, no result. |
| `Func<TResult>` | `TResult Method()` | No parameters, returns a result. |
| `Func<T1, ..., T16, TResult>` | `TResult Method(T1, ...)` | Parameters and a result. |
| `Predicate<T>` | `bool Method(T)` | Tests one value; similar in signature to `Func<T, bool>`. |

```csharp
using System;

public static class BuiltInDelegateDemo
{
    public static void Run()
    {
        Func<decimal, decimal> discountRule = basePrice => basePrice * 0.90m;
        Action<string> logAction = message => Console.WriteLine(message);
        Predicate<int> isEven = number => number % 2 == 0;

        Console.WriteLine(discountRule(100.00m));
        logAction("Discount calculated.");
        Console.WriteLine(isEven(42));
    }
}
```

### 2.3 Multicast Delegates

A delegate can hold an **invocation list** of several compatible methods. `+=` adds a handler and `-=` removes a matching handler. Invocation proceeds in list order.

```csharp
using System;

public static class MulticastDelegateDemo
{
    public static void Run()
    {
        Action<string> pipeline = LogToConsole;
        pipeline += LogToFile;       // Demonstration only: this writes to the console too.
        pipeline += LogToAuditTrail; // Demonstration only.

        pipeline("System started.");
        pipeline -= LogToFile;
    }

    private static void LogToConsole(string message) => Console.WriteLine($"Console: {message}");
    private static void LogToFile(string message) => Console.WriteLine($"File: {message}");
    private static void LogToAuditTrail(string message) => Console.WriteLine($"Audit: {message}");
}
```

Important multicast behavior:

- For a delegate with a non-`void` result, a normal invocation returns only the **last** method's result; earlier return values aren't collected.
- If a handler throws an exception that it doesn't catch, invocation stops and the exception propagates; later handlers aren't called.
- A delegate created for an instance method retains a reference to its target object. Lambdas can also capture local state. Long-lived publishers and subscriptions therefore need deliberate lifetime management; the `event` keyword helps control who can invoke the delegate.

### 2.4 Delegate Targets

A delegate can refer to a static method, an instance method, a lambda expression, an anonymous method, or a local function. For an instance method the delegate retains the target object; a lambda that captures variables retains the captured state for as long as the delegate remains reachable.

## 3. Delegates as Method Parameters (Higher-Order Functions)

A **higher-order function** accepts a delegate, returns a delegate, or both. That lets the caller supply part of an algorithm—for example, the operation to run and a policy that decides which exceptions are retryable.

```csharp
#nullable enable
using System;

public static class Retrier
{
    // maxAttempts counts total calls, including the initial call.
    public static T Execute<T>(
        Func<T> operation,
        int maxAttempts,
        Func<Exception, bool> shouldRetry)
    {
        ArgumentNullException.ThrowIfNull(operation);
        ArgumentNullException.ThrowIfNull(shouldRetry);

        if (maxAttempts <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(maxAttempts));
        }

        Exception? lastException = null;

        for (int attempt = 1; ; attempt++)
        {
            try
            {
                return operation();
            }
            catch (Exception ex) when (shouldRetry(ex))
            {
                lastException = ex;
                if (attempt == maxAttempts)
                {
                    break;
                }
            }
        }

        throw new InvalidOperationException(
            $"Operation failed after {maxAttempts} attempts.",
            lastException);
    }
}
```

The exception predicate is deliberately supplied by the caller rather than silently retrying every exception. This small synchronous example has no delay, cancellation, or backoff; the async backend example below adds those concerns.

## 4. Basic Syntax Example

This example combines a named validation delegate, `Action<T>` notifications, `Func<...>` arithmetic, and a method that accepts a sequence of delegates.

```csharp
#nullable enable
using System;
using System.Collections.Generic;

namespace MyBackendApp.Basics;

public static class DelegateDemo
{
    public delegate bool ValidationRule(string input);

    public static bool IsNotEmpty(string input) => !string.IsNullOrWhiteSpace(input);
    public static bool IsShortEnough(string input) => input is not null && input.Length <= 50;

    public static void Run()
    {
        ValidationRule rule = IsNotEmpty;
        Console.WriteLine(rule("hello")); // True

        Action<string> notifyPipeline = SendEmail;
        notifyPipeline += SendSms;
        notifyPipeline += LogNotification;
        notifyPipeline("Order shipped!");

        Func<int, int, int> add = (a, b) => a + b;
        Console.WriteLine(add(3, 4)); // 7

        ValidationRule[] rules = [IsNotEmpty, IsShortEnough];
        bool allValid = ValidateAll("Test Input", rules);
        Console.WriteLine(allValid);
    }

    private static bool ValidateAll(string input, IEnumerable<ValidationRule> rules)
    {
        ArgumentNullException.ThrowIfNull(input);
        ArgumentNullException.ThrowIfNull(rules);

        foreach (ValidationRule validationRule in rules)
        {
            if (!validationRule(input))
            {
                return false;
            }
        }

        return true;
    }

    private static void SendEmail(string message) => Console.WriteLine($"Email: {message}");
    private static void SendSms(string message) => Console.WriteLine($"SMS: {message}");
    private static void LogNotification(string message) => Console.WriteLine($"Log: {message}");
}
```

The notification delegate is an `Action<string>` multicast delegate. If one notification handler throws, the remaining handlers in that invocation list won't run unless the caller handles exceptions per handler.

## 5. Backend Examples: Resilience and Pipeline Composition

The examples are split into separate files. This avoids placing multiple file-scoped namespace declarations (`namespace Name;`) in a single C# source file, which is invalid.

### 5.1 Retry/Resilience Demonstration — `ResilientExecutor.cs`

This is a compact teaching example, not a complete production resilience policy. It bounds attempts, asks a delegate which exceptions are retryable, propagates cancellation, and lets a callback report retries.

```csharp
#nullable enable
using System;
using System.Threading;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.Resilience;

public sealed class ResilientExecutor
{
    private readonly int _maxAttempts;
    private readonly TimeSpan _delayBetweenRetries;

    public ResilientExecutor(int maxAttempts, TimeSpan delayBetweenRetries)
    {
        if (maxAttempts <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(maxAttempts));
        }

        if (delayBetweenRetries < TimeSpan.Zero)
        {
            throw new ArgumentOutOfRangeException(nameof(delayBetweenRetries));
        }

        _maxAttempts = maxAttempts;
        _delayBetweenRetries = delayBetweenRetries;
    }

    public async Task<T> ExecuteWithRetryAsync<T>(
        Func<CancellationToken, Task<T>> operation,
        Func<Exception, bool> shouldRetry,
        Action<Exception, int>? onRetry = null,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(operation);
        ArgumentNullException.ThrowIfNull(shouldRetry);

        Exception? lastException = null;

        for (int attempt = 1; ; attempt++)
        {
            cancellationToken.ThrowIfCancellationRequested();

            try
            {
                return await operation(cancellationToken).ConfigureAwait(false);
            }
            catch (Exception ex) when (
                ex is not OperationCanceledException && shouldRetry(ex))
            {
                lastException = ex;
                if (attempt == _maxAttempts)
                {
                    break;
                }

                onRetry?.Invoke(ex, attempt);
                await Task.Delay(_delayBetweenRetries, cancellationToken).ConfigureAwait(false);
            }
        }

        throw new InvalidOperationException(
            $"Operation failed after {_maxAttempts} attempts.",
            lastException);
    }
}
```

Retries can repeat side effects, so retry only transient failures and only when the operation is safe to repeat or protected by an idempotency strategy. Production policies usually also need exponential backoff with jitter, timeouts, telemetry, and service-specific handling (such as `Retry-After`). For modern .NET applications, evaluate `Microsoft.Extensions.Resilience` or `Microsoft.Extensions.Http.Resilience` rather than treating this small custom loop as a replacement.

### 5.2 A Small Delegate Pipeline — `SimplePipelineBuilder.cs`

This custom pipeline illustrates delegate composition. It is **not** ASP.NET Core's actual middleware implementation: ASP.NET Core's `RequestDelegate` takes `HttpContext` and returns `Task`.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.Pipeline;

public sealed class RequestContext
{
    public string Path { get; init; } = string.Empty;
    public Dictionary<string, object?> Items { get; } = new();
    public bool IsHandled { get; set; }
}

public delegate Task PipelineDelegate(RequestContext context);
public delegate PipelineDelegate MiddlewareComponent(PipelineDelegate next);

public sealed class SimplePipelineBuilder
{
    private readonly List<MiddlewareComponent> _components = [];

    public SimplePipelineBuilder Use(MiddlewareComponent middleware)
    {
        ArgumentNullException.ThrowIfNull(middleware);
        _components.Add(middleware);
        return this;
    }

    public PipelineDelegate Build()
    {
        PipelineDelegate pipeline = context =>
        {
            context.IsHandled = true;
            Console.WriteLine($"[Terminal] Request to '{context.Path}' reached the end.");
            return Task.CompletedTask;
        };

        // Compose in reverse so the first registered component executes first.
        for (int i = _components.Count - 1; i >= 0; i--)
        {
            pipeline = _components[i](pipeline);
        }

        return pipeline;
    }
}
```

### 5.3 Configure and Run the Pipeline — `PipelineDemo.cs`

```csharp
using System;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.Pipeline;

public static class PipelineDemo
{
    public static async Task RunAsync()
    {
        var builder = new SimplePipelineBuilder();

        builder.Use(next => async context =>
        {
            Console.WriteLine($"[Logging] Incoming request: {context.Path}");
            await next(context);
            Console.WriteLine($"[Logging] Completed request: {context.Path}");
        });

        builder.Use(next => async context =>
        {
            if (context.Path.StartsWith("/admin", StringComparison.OrdinalIgnoreCase))
            {
                Console.WriteLine("[Auth] Blocked: insufficient permissions.");
                context.IsHandled = true;
                return; // Short-circuits: the next delegate isn't invoked.
            }

            await next(context);
        });

        PipelineDelegate pipeline = builder.Build();
        await pipeline(new RequestContext { Path = "/orders" });
        Console.WriteLine("---");
        await pipeline(new RequestContext { Path = "/admin/settings" });
    }
}
```

The builder wraps the terminal delegate from last-registered component to first-registered component. At execution time, logging runs first; the authorization step can either call `next` or short-circuit the rest of the chain. The custom builder itself is intended for learning, not as a replacement for ASP.NET Core middleware or its dependency-injection and error-handling facilities.

## 6. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Delegate** | A type-safe object referring to compatible method(s). | Passes behavior as data for callbacks, policies, and pipelines. |
| **`Action<T>`** | A built-in delegate with inputs and no return value. | Fits notifications and other side-effecting callbacks. |
| **`Func<T, TResult>`** | A built-in delegate with inputs and a return value. | Fits predicates, transformations, and operations. |
| **Multicast delegate** | A delegate with an invocation list of multiple compatible handlers. | Supports ordered notification and pipeline-like calls; exceptions stop later handlers. |
| **Delegate target** | The instance method's target object, or captured state held by a closure. | A long-lived delegate can keep objects alive. |
| **Higher-order function** | A method that accepts or returns delegates. | Lets callers provide behavior such as a retry predicate or transformation. |
| **Invocation list** | The ordered methods called by a multicast delegate. | Determines handler order and which return value is observed. |
| **`RequestDelegate`** | ASP.NET Core's `Task`-returning delegate over `HttpContext`. | A framework building block for HTTP request processing. |

## 7. Official References

- [Introduction to delegates and events — C#](https://learn.microsoft.com/en-us/dotnet/csharp/delegates-overview) — type-safe late binding and delegate use cases.
- [Using delegates — C# Programming Guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/using-delegates) — method groups, instance targets, multicast invocation, and callbacks.
- [`RequestDelegate` API for ASP.NET Core 10](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.requestdelegate?view=aspnetcore-10.0) — actual framework delegate signature.
- [Introduction to resilient app development — .NET](https://learn.microsoft.com/en-us/dotnet/core/resilience/) — current Microsoft resilience packages and strategy guidance.
- [Build resilient HTTP apps: key development patterns — .NET](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience) — HTTP resilience handler guidance.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — .NET support tracks; .NET 10 is LTS.
