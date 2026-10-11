Examples target **.NET 10 (LTS)** and **C# 14**. Pattern matching has grown across several C# releases: C# 7 introduced declaration-pattern matching, C# 8 added switch expressions, C# 9 added relational and logical patterns, C# 10 added extended property patterns, and C# 11 added list patterns.

## 1. Conceptual Foundation

### 1.1 What Is Pattern Matching?

Pattern matching tests whether a value has a particular **type, value, range, or structure**. When a pattern matches, it can also bind the value—or parts of it—to variables. C# patterns appear in `is` expressions, `case` labels in `switch` statements, and arms of `switch` expressions.

Instead of testing a type, casting, and then separately checking a property, a type and property pattern can express those conditions together:

```csharp
#nullable enable

namespace MyBackendApp.Basics;

public sealed record SampleCircle(double Radius);

public static class PatternIntroductionDemo
{
    public static SampleCircle? BeforePatterns(object? value)
    {
        if (value is SampleCircle)
        {
            var circle = (SampleCircle)value; // Cast is safe after the type test.
            if (circle.Radius > 0)
            {
                return circle;
            }
        }

        return null;
    }

    public static SampleCircle? WithPatterns(object? value)
    {
        return value is SampleCircle { Radius: > 0 } circle
            ? circle
            : null;
    }
}
```

The pattern only binds `positiveCircle` when the runtime value is a non-null `Circle` whose `Radius` is greater than zero. Pattern matching can make branching more declarative, but ordinary Boolean conditions remain useful when they are clearer.

## 2. Categories of Patterns

### 2.1 Declaration and Type Patterns

A **declaration pattern** checks the runtime type and declares a variable for the matched value. A **type pattern** checks the type without declaring a variable. Type and declaration patterns don't match `null`.

```csharp
using System;

namespace MyBackendApp.Basics;

public static class TypePatternDemo
{
    public static void Run(object value)
    {
        if (value is int number) // Declaration pattern: type test plus binding.
        {
            Console.WriteLine($"It's an int: {number}");
        }

        bool isString = value is string; // Type pattern: test only.
        Console.WriteLine($"Is a string: {isString}");
    }
}
```

### 2.2 Constant Pattern

A constant pattern tests equality with a compile-time constant. `is null` and `is not null` are also constant-pattern checks; unlike an overloaded `==`, `is null` does not call a user-defined equality operator.

```csharp
using System;

namespace MyBackendApp.Basics;

public static class ConstantPatternDemo
{
    public static void DescribeStatus(int statusCode)
    {
        if (statusCode is 200)
        {
            Console.WriteLine("OK");
        }
    }
}
```

### 2.3 Relational Patterns

Relational patterns compare the input with a constant using `<`, `<=`, `>`, or `>=`. Switch arms are evaluated in order, so put more specific conditions before broader ones.

```csharp
using System;

namespace MyBackendApp.Basics;

public static class RelationalPatternDemo
{
    public static string ClassifyAge(int age) => age switch
    {
        < 0 => "Invalid",
        < 13 => "Child",
        < 20 => "Teenager",
        < 65 => "Adult",
        _ => "Senior"
    };

    public static void Run() => Console.WriteLine(ClassifyAge(16));
}
```

### 2.4 Logical Patterns: `and`, `or`, and `not`

Logical patterns combine other patterns. The precedence is `not`, then `and`, then `or`; use parentheses when a more complex expression might be misread.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Basics;

public static class LogicalPatternDemo
{
    public static void Run()
    {
        int value = 75;
        string? input = "";

        bool isValidPercentage = value is >= 0 and <= 100;
        bool isNullOrEmpty = input is null or "";
        bool isNotNull = input is not null;

        Console.WriteLine($"Valid percent: {isValidPercentage}");
        Console.WriteLine($"Null or empty: {isNullOrEmpty}");
        Console.WriteLine($"Not null: {isNotNull}");
    }
}
```

### 2.5 Property Patterns

A property pattern checks one or more properties or fields. A nested property pattern checks a property of a property; a matching object can also be bound to a variable.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Orders;

public sealed record Customer(string Tier);
public sealed record Order(string Status, decimal Total, Customer? Customer);

public static class OrderPatternDemo
{
    public static void Describe(Order? order)
    {
        if (order is { Status: "Pending", Total: > 1_000m } highValuePending)
        {
            Console.WriteLine($"High-value pending order: {highValuePending.Total}");
        }

        if (order is { Customer: { Tier: "Gold" } } goldOrder)
        {
            Console.WriteLine($"Gold customer order: {goldOrder.Total}");
        }
    }
}
```

A property pattern also tests that the value it examines is non-null. The nested form above works in C# 8 and later. C# 10 added an equivalent **extended property pattern**, `{ Customer.Tier: "Gold" }`, which can be convenient for short paths.

### 2.6 Positional Patterns

A positional pattern deconstructs a value and matches the resulting components by position. Positional records generate a `Deconstruct` method, and other types can provide one manually.

```csharp
#nullable enable

namespace MyBackendApp.Geometry;

public sealed record Point(int X, int Y);

public static class PointPatternDemo
{
    public static string Describe(Point? point) => point switch
    {
        null => "No point",
        (0, 0) => "Origin",
        (var x, 0) => $"On the X-axis at {x}",
        (0, var y) => $"On the Y-axis at {y}",
        (var x, var y) when x == y => "On the diagonal",
        _ => "Somewhere else"
    };
}
```

The positions correspond to the order of the `Deconstruct` parameters. Use property patterns instead when named properties make the test easier to understand.

### 2.7 `var` Pattern

A `var` pattern always matches—including `null`—and binds the input to a variable of its compile-time type. It is useful when a switch arm needs a value for a `when` guard.

```csharp
#nullable enable

namespace MyBackendApp.Basics;

public static class VarPatternDemo
{
    public static string NormalizeOrDescribe(string? input) => input switch
    {
        var text when !string.IsNullOrWhiteSpace(text) => text.Trim(),
        _ => "(empty)"
    };
}
```

Because `text` can be `null`, the guard uses `string.IsNullOrWhiteSpace` before calling `Trim()`.

### 2.8 List Patterns (C# 11+)

List patterns match a sequence of elements, optionally with a slice pattern (`..`) for zero or more elements. They work with arrays and other supported sequence-like types that expose the required length/count and indexer members; they don't enumerate every arbitrary `IEnumerable<T>`.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Basics;

public static class ListPatternDemo
{
    public static string Describe(int[]? values) => values switch
    {
        null => "Null input",
        [] => "Empty",
        [var single] => $"Single element: {single}",
        [var first, .., var last] when first != last =>
            $"Starts with {first}, ends with {last}",
        _ => "Other sequence shape"
    };

    public static void Run()
    {
        int[] numbers = [1, 2, 3]; // Collection expression (C# 12+).
        Console.WriteLine(Describe(numbers));
    }
}
```

`[]` matches an empty sequence, `[var single]` matches exactly one element, and `[var first, .., var last]` matches a sequence with at least two elements. A list pattern can contain at most one slice pattern. Include a fallback arm when every other shape needs a defined result: switch expressions don't provide exhaustive-match warnings for incomplete list patterns.

### 2.9 Discard Pattern

In a switch expression, `_` is the **discard pattern**: it matches any remaining value, including `null`, without binding it to a variable. Put it after more specific arms. In an `is` expression or a `switch` statement, use a `var _` pattern or the statement's `default` section when you need a catch-all.

## 3. `switch` Expressions vs. `switch` Statements`

A `switch` statement executes statements in the first matching section. A `switch` expression selects and returns the value of the first matching arm, so it fits well in assignments, returns, and expression-bodied members.

```csharp
int httpStatusCode = 200;

// Traditional switch statement.
string result;
switch (httpStatusCode)
{
    case 200:
        result = "OK";
        break;
    case 404:
        result = "Not Found";
        break;
    default:
        result = "Unknown";
        break;
}

// Value-returning switch expression.
string result2 = httpStatusCode switch
{
    200 => "OK",
    404 => "Not Found",
    _ => "Unknown"
};
```

### 3.1 Exhaustiveness and Arm Order

A switch expression doesn't syntactically require a discard (`_`) arm. The compiler warns when it detects input values that aren't handled, but exhaustiveness analysis has limits. If no arm matches at runtime, .NET throws `SwitchExpressionException`. A discard arm is a useful fallback for open-ended inputs; it also catches values that weren't anticipated.

C# 14 doesn't treat an ordinary abstract class or record hierarchy as a closed set of alternatives. Therefore, a `_` arm on a base record prevents a runtime no-match, but it also means adding a new derived record won't by itself produce a compiler warning at the switch site. If every known subtype must be handled, use an analyzer, tests, or another project-level design that enforces that rule.

Arms are checked in text order. Put specific cases before general cases; an earlier unguarded pattern that covers a later arm makes that later arm unreachable and produces a compiler error.

## 4. Basic Syntax Example

This example combines type, property, relational, and discard patterns. It validates dimensions before calculating an area and uses a switch expression to summarize a list-shaped input.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Basics;

public abstract record Shape;
public sealed record Circle(double Radius) : Shape;
public sealed record Rectangle(double Width, double Height) : Shape;
public sealed record Triangle(double Base, double Height) : Shape;

public static class PatternMatchingDemo
{
    public static double CalculateArea(Shape shape) => shape switch
    {
        null => throw new ArgumentNullException(nameof(shape)),

        Circle { Radius: > 0 } circle when double.IsFinite(circle.Radius) =>
            Math.PI * circle.Radius * circle.Radius,
        Circle => throw new ArgumentOutOfRangeException(
            nameof(shape), "Circle radius must be positive and finite."),

        Rectangle { Width: > 0, Height: > 0 } rectangle
            when double.IsFinite(rectangle.Width) && double.IsFinite(rectangle.Height) =>
            rectangle.Width * rectangle.Height,
        Rectangle => throw new ArgumentOutOfRangeException(
            nameof(shape), "Rectangle dimensions must be positive and finite."),

        Triangle { Base: > 0, Height: > 0 } triangle
            when double.IsFinite(triangle.Base) && double.IsFinite(triangle.Height) =>
            0.5 * triangle.Base * triangle.Height,
        Triangle => throw new ArgumentOutOfRangeException(
            nameof(shape), "Triangle dimensions must be positive and finite."),

        _ => throw new NotSupportedException(
            $"Unknown shape type: {shape.GetType().Name}")
    };

    public static void Run()
    {
        Shape[] shapes =
        [
            new Circle(5),
            new Rectangle(4, 6),
            new Triangle(3, 8)
        ];

        foreach (Shape shape in shapes)
        {
            Console.WriteLine($"{shape.GetType().Name}: Area = {CalculateArea(shape):F2}");
        }

        int[]? scores = [95, 82, 77];
        string summary = scores switch
        {
            null => "Scores unavailable",
            [] => "No scores recorded",
            [var only] => $"Single score: {only}",
            [var first, .., var last] when first != last =>
                $"First: {first}, Last: {last}, Count: {scores.Length}",
            _ => "Other score sequence"
        };

        Console.WriteLine(summary);
    }
}
```

## 5. Backend Examples: Typed Payment Events and Gateway Results

These examples use separate files. A file-scoped namespace declaration applies to one source file, so separate namespaces belong in separate `.cs` files (or use block-scoped namespace syntax).

### 5.1 Payment Event Types — `PaymentEvent.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Domain.Payments;

public abstract record PaymentEvent;
public sealed record PaymentAuthorized(string TransactionId, decimal Amount) : PaymentEvent;
public sealed record PaymentDeclined(string ReasonCode) : PaymentEvent;
public sealed record PaymentRefunded(
    string TransactionId,
    decimal Amount,
    DateTimeOffset RefundedAtUtc) : PaymentEvent;
public sealed record PaymentDisputed(string TransactionId, string DisputeReason) : PaymentEvent;
```

### 5.2 Payment Event Routing — `PaymentEventRouter.cs`

A switch expression can route domain events and extract values directly from their record properties. Specific property patterns appear before the broader type patterns.

```csharp
#nullable enable
using System;
using MyBackendApp.Core.Domain.Payments;

namespace MyBackendApp.Core.Application.Payments;

public sealed record NotificationPlan(
    string Channel,
    string MessageTemplate,
    bool RequiresManualReview);

public static class PaymentEventRouter
{
    public static NotificationPlan DetermineNotificationPlan(PaymentEvent paymentEvent)
    {
        ArgumentNullException.ThrowIfNull(paymentEvent);

        return paymentEvent switch
        {
            PaymentAuthorized { Amount: > 10_000m } large =>
                new NotificationPlan(
                    "Email+SMS",
                    $"Large payment authorized: {large.Amount:C}",
                    RequiresManualReview: true),

            PaymentAuthorized authorized =>
                new NotificationPlan(
                    "Email",
                    $"Payment authorized: {authorized.Amount:C}",
                    RequiresManualReview: false),

            PaymentDeclined { ReasonCode: "INSUFFICIENT_FUNDS" } =>
                new NotificationPlan(
                    "Email",
                    "Payment declined due to insufficient funds.",
                    RequiresManualReview: false),

            PaymentDeclined declined =>
                new NotificationPlan(
                    "Email",
                    $"Payment declined: {declined.ReasonCode}",
                    RequiresManualReview: false),

            PaymentRefunded refund when refund.Amount > 5_000m =>
                new NotificationPlan(
                    "Email+SMS",
                    $"Large refund processed: {refund.Amount:C}",
                    RequiresManualReview: true),

            PaymentRefunded refund =>
                new NotificationPlan(
                    "Email",
                    $"Refund processed: {refund.Amount:C}",
                    RequiresManualReview: false),

            PaymentDisputed dispute =>
                new NotificationPlan(
                    "Email+SMS",
                    $"Dispute raised: {dispute.DisputeReason}",
                    RequiresManualReview: true),

            _ => throw new NotSupportedException(
                $"Unhandled payment event type: {paymentEvent.GetType().Name}")
        };
    }
}
```

The final `_` arm is a defensive runtime fallback, not compile-time exhaustiveness for the abstract record hierarchy. The `:C` formatting is illustrative; production notifications should use an explicit currency and localization policy.

### 5.3 Gateway Response Classification — `GatewayResponseHandler.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Api.Controllers.Support;

public abstract record GatewayResult;
public sealed record GatewaySuccess(string TransactionId) : GatewayResult;
public sealed record GatewayClientError(int StatusCode, string Message) : GatewayResult;
public sealed record GatewayServerError(int StatusCode) : GatewayResult;

public static class GatewayResponseHandler
{
    public static string DescribeOutcome(GatewayResult result)
    {
        ArgumentNullException.ThrowIfNull(result);

        return result switch
        {
            GatewaySuccess success =>
                $"Success: transaction {success.TransactionId} completed.",

            GatewayClientError { StatusCode: 429 } =>
                "Gateway rate limit reached; apply the configured retry/backoff policy.",

            GatewayClientError error =>
                $"Gateway client error ({error.StatusCode}): {error.Message}",

            GatewayServerError { StatusCode: >= 500 and < 600 } serverError =>
                $"Gateway server error ({serverError.StatusCode}); escalate to on-call.",

            GatewayServerError serverError =>
                $"Unexpected gateway server status: {serverError.StatusCode}.",

            _ => throw new NotSupportedException(
                $"Unhandled gateway result type: {result.GetType().Name}")
        };
    }
}
```

In production, keep transport-level status handling separate from retry execution: honor provider retry hints such as `Retry-After`, bound retries, and make non-idempotent payment operations safe to retry.

## 6. Key Terms Summary

| Term | Definition | Purpose in backend development |
|---|---|---|
| **Pattern matching** | Testing a value against a type, constant, range, or structural pattern. | Makes branching over domain values and typed results easier to read. |
| **Declaration pattern** | A type test that also binds the matching value to a variable. | Narrows a value safely without a separate cast. |
| **Type pattern** | Tests the runtime type without declaring a variable. | Selects behavior by type when the matched value itself isn't needed. |
| **Constant pattern** | Tests whether a value equals a constant, such as `200` or `null`. | Classifies fixed status codes, enum values, and nulls. |
| **Relational pattern** | Compares a value with a constant using `<`, `<=`, `>`, or `>=`. | Expresses ranges and thresholds directly in a pattern. |
| **Logical pattern** | Combines patterns with `and`, `or`, and `not`. | Builds compact Boolean constraints from simpler patterns. |
| **Property pattern** | Matches selected properties or fields, including nested paths. | Checks domain-object state without manual nested conditionals. |
| **Positional pattern** | Matches values exposed through `Deconstruct` in positional order. | Concisely matches records, tuples, or other deconstructable values. |
| **`var` pattern** | Always matches and binds the input value. | Supplies a variable for further checks, often in a `when` guard. |
| **Discard pattern** | `_` matches a value without binding it. | Provides a catch-all arm in switch expressions. |
| **List pattern** | Matches sequence length and element patterns, with optional `..` slice. | Validates small sequence shapes without manual index checks. |
| **Switch expression** | An expression that returns the result of the first matching arm. | Keeps classification and mapping logic concise; review coverage and fallback behavior. |
| **Exhaustiveness** | Whether all possible inputs are handled by the patterns. | Helps prevent missed cases, while remembering C# 14's analysis has limits. |

## 7. Official References

- [Pattern matching overview — C#](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/patterns/pattern-matching) — pattern contexts, binding, arm order, and general guidance.
- [Patterns — C# language reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns) — type, constant, relational, logical, property, positional, `var`, discard, and list patterns.
- [`switch` expression — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/switch-expression) — arms, guards, subsumption, and non-exhaustive behavior.
- [C# language version history](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-version-history) — language feature versions, including C# 11 list patterns and C# 14.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — support tracks; .NET 10 is LTS.
