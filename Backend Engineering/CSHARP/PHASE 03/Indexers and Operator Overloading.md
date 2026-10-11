Examples target **.NET 10 (LTS)** and **C# 14**. Indexers give instances collection-like access through `[]`; operator overloading gives value-like types natural operator syntax where the meaning is clear.

## Part 1: Indexers

### 1.1 Concept

An **indexer** is a parameterized property. It lets callers read or write a value with syntax such as `collection[key]` instead of calling methods like `GetItem(key)` and `SetItem(key, value)`. The type decides what the index means and where the value comes from; the implementation might use an array, dictionary, database page, or a computed result.

```csharp
public class SomeCollection
{
    public ReturnType this[ParameterType index]
    {
        get { /* return the indexed value */ }
        set { /* assign value; use the implicit `value` parameter */ }
    }
}
```

### 1.2 Key Rules

- `this` declares the indexer; an indexer doesn't have a separate member name.
- Indexers are **instance members**, not static members.
- The index parameter can be any appropriate type—not only `int`—and an indexer can take more than one parameter.
- You can overload indexers when their parameter lists differ. They can't be distinguished only by return type.
- An indexer can be read-only (`get`), read/write (`get` and `set`), or technically write-only (`set`); write-only designs are uncommon and often confusing.
- The `set` accessor receives the assigned item through the implicit `value` parameter.
- C# 9 and later allow an `init` accessor for values that may be set during object initialization but not changed afterward.
- Indexers aren't automatically implemented properties: provide the storage or lookup logic yourself.

### 1.3 Basic Example: Weekly Schedule

The schedule uses a nullable element type because a day may have no entry yet. Both accessors validate the index consistently.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Basics;

public sealed class WeekSchedule
{
    private const int DayCount = 7;
    private readonly string?[] _days = new string?[DayCount];

    public string? this[int dayIndex]
    {
        get
        {
            ValidateDayIndex(dayIndex);
            return _days[dayIndex];
        }
        set
        {
            ValidateDayIndex(dayIndex);
            _days[dayIndex] = value;
        }
    }

    private static void ValidateDayIndex(int dayIndex)
    {
        if (dayIndex < 0 || dayIndex >= DayCount)
        {
            throw new ArgumentOutOfRangeException(
                nameof(dayIndex), "Day index must be between 0 and 6.");
        }
    }
}

public static class WeekScheduleDemo
{
    public static void Run()
    {
        var schedule = new WeekSchedule();
        schedule[0] = "Team standup";
        schedule[1] = "Code review";

        Console.WriteLine(schedule[0]); // Team standup
        Console.WriteLine(schedule[1]); // Code review
    }
}
```

### 1.4 Multi-Parameter Indexer: Seat Map

An indexer can take multiple parameters, which is convenient for a two-dimensional abstraction. Validate bounds explicitly so callers receive an argument-related exception instead of an array implementation detail.

```csharp
using System;

namespace MyBackendApp.Basics;

public sealed class SeatMap
{
    private readonly bool[,] _occupied;

    public SeatMap(int rows, int columns)
    {
        if (rows <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(rows), "Rows must be positive.");
        }

        if (columns <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(columns), "Columns must be positive.");
        }

        _occupied = new bool[rows, columns];
    }

    public bool this[int row, int column]
    {
        get
        {
            ValidateCoordinates(row, column);
            return _occupied[row, column];
        }
        set
        {
            ValidateCoordinates(row, column);
            _occupied[row, column] = value;
        }
    }

    private void ValidateCoordinates(int row, int column)
    {
        if (row < 0 || row >= _occupied.GetLength(0))
        {
            throw new ArgumentOutOfRangeException(nameof(row));
        }

        if (column < 0 || column >= _occupied.GetLength(1))
        {
            throw new ArgumentOutOfRangeException(nameof(column));
        }
    }
}
```

Usage looks like `seatMap[3, 7] = true;`. The indexer accepts two separate integer arguments; this differs from passing a single tuple key such as `seatMap[(3, 7)]`.

### 1.5 `init`-Only Indexer

An `init` accessor can make an indexer assignable only during object construction. This is useful for fixed configuration data, not for a collection that callers are expected to keep changing.

```csharp
#nullable enable
using System.Collections.Generic;

namespace MyBackendApp.Basics;

public sealed class StartupOptions
{
    private readonly Dictionary<string, string> _values = new();

    public string? this[string key]
    {
        get => _values.TryGetValue(key, out string? value) ? value : null;
        init => _values[key] = value;
    }
}

public static class StartupOptionsDemo
{
    public static void Run()
    {
        var options = new StartupOptions
        {
            ["mode"] = "safe"
        };

        Console.WriteLine(options["mode"]); // safe
        // options["mode"] = "fast"; // Not allowed after initialization.
    }
}
```

## Part 2: Operator Overloading

### 2.1 Concept and Design Guidance

**Operator overloading** defines how built-in operators work when one or both operands are instances of a user-defined type. It is appropriate when the operation has an obvious, conventional meaning—such as adding vectors or adding amounts in the same currency.

Use it sparingly. A `+` operator for a value-like `Money` type can be intuitive; a `+` operator for a `Customer` usually isn't. Define behavior for invalid combinations (such as different currencies) explicitly, and keep it consistent with the type's named methods and equality semantics.

### 2.2 Declaration Rules in C# 14

For the usual unary and binary operator overloads, declare a `public static` method with the `operator` keyword. At least one operand must have the containing type (or its nullable form). The method's parameter count must match the operator's arity.

C# 14 adds two important exceptions to the traditional `public static` rule:

| Operator form | C# 14 declaration |
|---|---|
| Traditional unary/binary operators such as `+`, `-`, `==`, and `<` | `public static` operator method. |
| Compound assignments such as `+=` and `-=` | May be overloaded as a `public` **instance** `void` operator with one right-hand-side parameter. If no custom compound operator exists, existing compound-assignment behavior can use the corresponding binary operator. |
| `++` and `--` | Can be traditional static operators or C# 14 `public` **instance** `void` operators with no parameters. |

The familiar static form looks like this:

```csharp
public static ReturnType operator +(Type left, Type right)
{
    // Return the result of adding left and right.
}
```

### 2.3 Paired and Non-Overloadable Operators

The compiler requires these pairs:

| If you overload… | You must also overload… |
|---|---|
| `==` | `!=` |
| `<` | `>` |
| `<=` | `>=` |

If you overload `==` and `!=`, also keep them consistent with `Equals` and `GetHashCode`; the compiler may warn when the equality members are missing. The operators should agree on null behavior and value semantics.

You can't directly overload `&&` or `||`. A type can define conditional logical behavior by overloading `&` or `|` along with both `true` and `false` operators; that is an advanced pattern, not a substitute for a clear Boolean API. Assignment (`=`), member access (`.`), null-conditional (`?.`), null-coalescing (`??`), and type-test operators such as `is` also can't be overloaded.

### 2.4 User-Defined Conversions

A type can define `implicit` or `explicit` conversions when the conversion is part of its domain. Use an **implicit** conversion only when it is safe, predictable, and won't lose information or throw. Use an **explicit** conversion when the caller should see that a potentially lossy or failing conversion is occurring.

## Part 3: Basic Operator Example — `Vector2D`

A two-dimensional vector is a natural use for `+` and `-`. The `readonly struct` keeps its value state immutable; its default value `(0, 0)` is also meaningful.

```csharp
using System;

namespace MyBackendApp.Basics;

public readonly struct Vector2D
{
    public double X { get; }
    public double Y { get; }

    public Vector2D(double x, double y)
    {
        X = x;
        Y = y;
    }

    public static Vector2D operator +(Vector2D left, Vector2D right) =>
        new(left.X + right.X, left.Y + right.Y);

    public static Vector2D operator -(Vector2D left, Vector2D right) =>
        new(left.X - right.X, left.Y - right.Y);

    public override string ToString() => $"({X}, {Y})";
}

public static class Vector2DDemo
{
    public static void Run()
    {
        var first = new Vector2D(1, 2);
        var second = new Vector2D(3, 4);
        Vector2D sum = first + second;
        Vector2D difference = second - first;

        Console.WriteLine(sum);        // (4, 6)
        Console.WriteLine(difference); // (2, 2)
    }
}
```

## Part 4: Backend Examples

### 4.1 Indexer over an In-Process Feature-Flag Store

An indexer can expose a small, synchronous key-to-value lookup without exposing its backing dictionary. This example returns `false` for a missing key and uses case-insensitive keys.

The interface belongs in an application/core abstraction; the infrastructure implementation belongs in a separate file and namespace.

#### `IFeatureFlagStore.cs`

```csharp
#nullable enable

namespace MyBackendApp.Core.Abstractions;

public interface IFeatureFlagStore
{
    bool this[string featureKey] { get; }
}
```

#### `InMemoryFeatureFlagStore.cs`

```csharp
#nullable enable
using System;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Logging;
using MyBackendApp.Core.Abstractions;

namespace MyBackendApp.Infrastructure.Caching;

public sealed class InMemoryFeatureFlagStore : IFeatureFlagStore
{
    private static readonly TimeSpan CacheLifetime = TimeSpan.FromMinutes(5);
    private readonly IMemoryCache _cache;
    private readonly ILogger<InMemoryFeatureFlagStore> _logger;

    public InMemoryFeatureFlagStore(
        IMemoryCache cache,
        ILogger<InMemoryFeatureFlagStore> logger)
    {
        ArgumentNullException.ThrowIfNull(cache);
        ArgumentNullException.ThrowIfNull(logger);
        _cache = cache;
        _logger = logger;
    }

    public bool this[string featureKey]
    {
        get
        {
            string key = CreateCacheKey(featureKey);
            bool enabled = _cache.TryGetValue(key, out bool cachedValue) && cachedValue;
            _logger.LogDebug("Feature flag {FeatureKey} evaluated to {Enabled}", featureKey, enabled);
            return enabled;
        }
        set
        {
            string key = CreateCacheKey(featureKey);
            _cache.Set(key, value, CacheLifetime);
            _logger.LogInformation("Feature flag {FeatureKey} set to {Enabled}", featureKey, value);
        }
    }

    private static string CreateCacheKey(string? featureKey)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(featureKey);
        return $"feature-flag:{featureKey.Trim().ToUpperInvariant()}";
    }
}
```

#### `CheckoutService.cs`

```csharp
#nullable enable
using System;
using MyBackendApp.Core.Abstractions;

namespace MyBackendApp.Core.Services;

public sealed class CheckoutService
{
    private readonly IFeatureFlagStore _featureFlags;

    public CheckoutService(IFeatureFlagStore featureFlags)
    {
        ArgumentNullException.ThrowIfNull(featureFlags);
        _featureFlags = featureFlags;
    }

    public decimal ApplyDiscount(decimal total)
    {
        if (_featureFlags["NewDiscountEngine"])
        {
            return total * 0.9m;
        }

        return total;
    }
}
```

This is deliberately an **in-process** cache: each value expires five minutes after it is set, and a missing or expired flag reads as `false`. Register `IMemoryCache` with `services.AddMemoryCache()`. Separate application instances don't share this cache, and it has no remote refresh or invalidation; production feature-flag systems need an appropriate shared provider and update/expiry policy. Keep cache keys to a controlled set and set appropriate size/expiry limits rather than caching arbitrary user input. The indexer is synchronous, so don't hide remote or asynchronous I/O in its getter. Logging every read may also be too noisy at high request volume; choose telemetry levels and sampling deliberately. `IMemoryCache` and `ILogger<T>` are available through the ASP.NET Core shared framework or their respective Microsoft.Extensions packages.

## Part 5: Financial Value Object Example — `Money`

Operator overloads can make arithmetic and comparisons readable, but they don't make currency safety a compile-time guarantee. This example checks currency compatibility at runtime and uses a named invariant-checking helper. It is an immutable **value object implemented as a sealed class**, which avoids an invalid `default(Money)` struct state; a struct can always be default-initialized.

### `Money.cs`

```csharp
#nullable enable
using System;
using System.Globalization;

namespace MyBackendApp.Core.Domain;

public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        Amount = amount;
        Currency = currency.Trim().ToUpperInvariant();
    }

    private static void EnsureSameCurrency(Money left, Money right)
    {
        ArgumentNullException.ThrowIfNull(left);
        ArgumentNullException.ThrowIfNull(right);

        if (!StringComparer.Ordinal.Equals(left.Currency, right.Currency))
        {
            throw new InvalidOperationException(
                $"Cannot operate on different currencies: {left.Currency} vs {right.Currency}.");
        }
    }

    public static Money operator +(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    public static Money operator -(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount - right.Amount, left.Currency);
    }

    public static Money operator *(Money money, decimal factor)
    {
        ArgumentNullException.ThrowIfNull(money);
        return new Money(money.Amount * factor, money.Currency);
    }

    public static Money operator *(decimal factor, Money money) => money * factor;

    public static bool operator ==(Money? left, Money? right)
    {
        if (ReferenceEquals(left, right))
        {
            return true;
        }

        return left is not null && left.Equals(right);
    }

    public static bool operator !=(Money? left, Money? right) => !(left == right);

    public static bool operator >(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return left.Amount > right.Amount;
    }

    public static bool operator <(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return left.Amount < right.Amount;
    }

    public bool Equals(Money? other) =>
        other is not null
        && Amount == other.Amount
        && StringComparer.Ordinal.Equals(Currency, other.Currency);

    public override bool Equals(object? obj) => obj is Money other && Equals(other);

    public override int GetHashCode() =>
        HashCode.Combine(Amount, StringComparer.Ordinal.GetHashCode(Currency));

    public override string ToString() =>
        $"{Amount.ToString(CultureInfo.InvariantCulture)} {Currency}";
}
```

### `InvoiceService.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using MyBackendApp.Core.Domain;

namespace MyBackendApp.Core.Services;

public sealed class InvoiceService
{
    public Money CalculateTotal(string currency, IEnumerable<Money> lineItems)
    {
        ArgumentNullException.ThrowIfNull(lineItems);

        Money total = new(0m, currency);
        foreach (Money item in lineItems)
        {
            total += item; // Uses the overloaded binary + operator.
        }

        return total;
    }

    public bool ExceedsCreditLimit(Money total, Money creditLimit) => total > creditLimit;
}
```

Addition, subtraction, and ordering comparisons throw when currencies differ; `==` and `!=` instead compare both amount and normalized currency. This sample validates that a currency string is present and normalizes its casing, but it doesn't validate against the ISO 4217 registry, perform currency conversion, or apply currency-specific rounding. A production system should use a controlled currency-code type and explicit rounding rules. The example intentionally omits `IComparable<Money>`: sorting across different currencies needs an explicit ordering or conversion policy, while its `>` and `<` operators are defined only for matching currencies. Because `Money` is a class, callers can still pass `null` at runtime; arithmetic/comparison helpers guard against that, and the equality operators handle null consistently.

## 6. Key Terms Summary

| Term | Meaning |
|---|---|
| **Indexer** | A parameterized property that provides `instance[key]` syntax. |
| **Indexer parameter** | The key or keys inside `[]`; the types depend on the abstraction. |
| **Multi-parameter indexer** | An indexer with more than one argument, such as `this[int row, int column]`. |
| **`get` / `set` / `init` accessor** | Read, write, or initialization-only behavior for an indexer. |
| **Operator overloading** | Defining how operators behave for operands of a user-defined type. |
| **Paired operators** | Required pairs such as `==`/`!=` and `<`/`>`. |
| **User-defined conversion** | An `implicit` or `explicit` conversion declared by a source or target type. |
| **Value object** | An immutable object whose meaning is determined by its data rather than identity. |
| **C# 14 compound operator** | An instance `void` overload such as `operator +=`, available in C# 14 and later. |

## 7. Official References

- [Indexers — C# programming guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/indexers/) — indexer declarations, overloads, dictionaries, and multi-parameter examples.
- [`static` modifier — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/static) — confirms that `static` can't be applied to an indexer.
- [`init` keyword — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/init) — init-only accessors for properties and indexers.
- [Operator overloading — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading) — overloadable operators, pairing rules, and C# 14 additions.
- [User-defined conversion operators — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/user-defined-conversion-operators) — when to use implicit or explicit conversions.
- [Operator overloads — .NET design guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/operator-overloads) — use operators cautiously and keep their meanings intuitive.
- [In-memory caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-10.0) — `IMemoryCache`, expiration, DI registration, and cache-size guidelines.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — .NET 10 is LTS.
