Examples target **.NET 10 (LTS)** and **C# 14**. Most samples use the classic `this`-parameter syntax, which remains supported; a short section also introduces C# 14 extension blocks.

## 1. Conceptual Foundation

### 1.1 What Is an Extension Method?

An **extension method** is a static method that C# lets you call with instance-method syntax on an existing type. It can add convenient operations to types you don't own—such as `string`, `IEnumerable<T>`, or a third-party type—without changing or recompiling that type's assembly and without using inheritance.

The original type isn't modified at runtime. For the classic extension-method syntax, the compiler binds a call such as `email.NormalizeEmail()` to a static method call like `StringExtensions.NormalizeEmail(email)`. Extension methods are resolved at compile time; they aren't virtual instance members.

| Call-site syntax | Equivalent static call |
|---|---|
| `var clean = email.NormalizeEmail();` | `var clean = StringExtensions.NormalizeEmail(email);` |

LINQ uses the same pattern: methods such as `Where`, `Select`, and `OrderBy` are extension methods. The `Enumerable` class defines operators for `IEnumerable<T>`; `Queryable` defines expression-tree-based operators for `IQueryable<T>`.

### 1.2 Classic Extension Methods and C# 14 Extension Blocks

Before C# 14, extension methods are declared as `static` methods inside a non-nested, non-generic static class. The first parameter uses `this` to identify the receiver type. C# 14 also supports **extension blocks**, declared in a static, non-nested, non-generic class, which can contain extension methods, properties, and operators. This guide focuses on extension methods; the classic form is still valid in C# 14 and is widely used.

```csharp
namespace MyBackendApp.CSharp14Extensions;

public static class StringExtensionMembers
{
    extension(string text)
    {
        public bool HasMeaningfulContent() => !string.IsNullOrWhiteSpace(text);
    }
}
```

With the namespace in scope, client code can call `text.HasMeaningfulContent()`. In an extension block, the receiver is named in `extension(string text)`; extension methods declared inside the block don't repeat the `this` parameter.

## 2. Defining and Calling a Classic Extension Method

Classic extension methods follow these rules:

1. Put the method in a `static` class that is non-nested and non-generic.
2. Declare the method `static` and make it accessible to the calling code.
3. Mark its first parameter with `this`; that parameter specifies the receiver type.
4. Ensure the calling project references the assembly/package containing the extension class, then import its namespace in the calling file (or make it available through a global using).
5. Call the method like an instance method, or call it directly as a static method and pass the receiver explicitly.

```csharp
using System;

namespace MyBackendApp.Extensions;

public static class StringExtensions
{
    public static string NormalizeEmail(this string input)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(input);
        return input.Trim().ToLowerInvariant();
    }

    public static bool IsWithinLength(this string input, int maxLength)
    {
        ArgumentNullException.ThrowIfNull(input);
        if (maxLength < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(maxLength));
        }

        return input.Length <= maxLength;
    }
}
```

The `this` parameter is supplied by the receiver in `input.IsWithinLength(30)`; callers pass only the remaining arguments. The same method can be called statically as `StringExtensions.IsWithinLength(input, 30)`.

> **Email policy note:** The sample's `Trim().ToLowerInvariant()` is an application-specific lookup policy, not a universal rule for normalizing email addresses. In particular, blindly lowercasing the entire address may not match the identity provider's or system's rules for the local part. Normalize identifiers according to the system that owns them.

## 3. Extension-Method Resolution and Null Receivers

### 3.1 Instance Members Take Precedence

When an applicable instance member exists on a type, it takes precedence over an extension method with the same name and signature. An extension method can't override, hide, or add virtual dispatch to a real member. If multiple in-scope extension methods are equally applicable, the call can be ambiguous; a static call using the containing class can make the intended method explicit.

Extension methods are only considered when the compiler doesn't find a suitable member on the receiver type. Importing a namespace makes its extension methods eligible for lookup; it doesn't change the type itself.

### 3.2 A Null Receiver Doesn't Throw by Itself

Because the classic extension call is compiled as a static call, the receiver can be `null` without an immediate `NullReferenceException`. The method still has to handle that value safely. Nullable annotations also matter: a nullable receiver should be reflected in the `this` parameter type.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Extensions;

public static class NullSafeStringExtensions
{
    public static bool IsNullOrEmpty(this string? input) =>
        string.IsNullOrEmpty(input);
}
```

```csharp
#nullable enable
using MyBackendApp.Extensions;

namespace MyBackendApp.Basics;

public static class NullSafeExtensionDemo
{
    public static bool Run()
    {
        string? value = null;
        return value.IsNullOrEmpty(); // Calls the extension with null; returns true.
    }
}
```

This is a convenient null-checking pattern, not automatic null safety for all extensions: an extension that dereferences its receiver without checking it can still throw.

## 4. Extensions on Interfaces and Generic Types

An extension method can target an interface or an open generic type. That lets one implementation work with many collections or types that implement the interface.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Extensions;

public static class EnumerableExtensions
{
    public static bool IsEmpty<T>(this IEnumerable<T> source)
    {
        ArgumentNullException.ThrowIfNull(source);
        return !source.Any();
    }

    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source)
        where T : class
    {
        ArgumentNullException.ThrowIfNull(source);
        return source
            .Where(item => item is not null)
            .Select(item => item!);
    }
}
```

`IsEmpty` may enumerate the source to determine whether it has an element. `WhereNotNull` returns a deferred sequence and uses nullable annotations to describe a source containing nullable references and a result containing only non-null `T` values.

> **Provider boundary:** These helpers are declared for `IEnumerable<T>`, so their internal `Any`, `Where`, and `Select` calls use LINQ-to-Objects delegates. If you pass an `IQueryable<T>`, the receiver can still convert to `IEnumerable<T>` and the helper may enumerate or filter on the client. Use provider-native operators—or write an overload that accepts `IQueryable<T>`—when the operation must remain in a database query.

### 4.1 Fluent Chaining

A **fluent API** lets callers chain operations by returning a type that supports the next operation. For query extensions, returning `IQueryable<T>` preserves the provider's query definition so more filters, sorts, or paging can be composed before execution. The method should not materialize the query unless materialization is explicitly its purpose.

## 5. When to Use Extension Methods

| Use extension methods when… | Prefer another design when… |
|---|---|
| Adding focused utility behavior to a type you don't own, using only its public API. | The logic needs private or protected state; put it on the type or behind an appropriate abstraction. |
| Organizing related `IServiceCollection` registrations into a named, reusable setup method. | The behavior is core domain logic that conceptually belongs on a domain type or domain service. |
| Adding generic helpers to an interface such as `IEnumerable<T>`. | Extensions become numerous, obscure, or surprising; callers may have trouble discovering them. |
| Composing provider-aware query helpers that return `IQueryable<T>`. | The helper silently materializes data, switches to local execution, or hides expensive work. |

Extension methods can provide reusable operations over an interface, but they do **not** add a member to that interface, change its contract, or provide polymorphic dispatch. A default interface member (available since C# 8) is a different mechanism for defining actual interface behavior.

## 6. Basic Syntax Example

The email check below deliberately tests only a **basic shape**; it isn't complete email-address validation. The truncation method treats `prefixLength` as the number of source characters to retain before appending an ellipsis.

### 6.1 `StringValidationExtensions.cs` and `DecimalExtensions.cs`

```csharp
using System;

namespace MyBackendApp.Extensions;

public static class StringValidationExtensions
{
    public static bool HasBasicEmailShape(this string? input)
    {
        if (string.IsNullOrWhiteSpace(input))
        {
            return false;
        }

        string candidate = input.Trim();
        int at = candidate.IndexOf('@');
        if (at <= 0 || at != candidate.LastIndexOf('@') || at == candidate.Length - 1)
        {
            return false;
        }

        string domain = candidate[(at + 1)..];
        return domain.Contains('.') && !domain.StartsWith('.') && !domain.EndsWith('.');
    }

    public static string Truncate(this string input, int prefixLength)
    {
        ArgumentNullException.ThrowIfNull(input);
        if (prefixLength < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(prefixLength));
        }

        return input.Length <= prefixLength
            ? input
            : input[..prefixLength] + "...";
    }
}

public static class DecimalExtensions
{
    public static decimal PercentOf(this decimal value, decimal percent) =>
        value * (percent / 100m);
}
```

`string` ranges count UTF-16 code units. If truncating user-visible text that may contain emoji or combining characters, use a text-element-aware strategy so a cut doesn't split a displayed character.

### 6.2 Calling the Extensions — `ExtensionMethodDemo.cs`

```csharp
using System;
using MyBackendApp.Extensions;

namespace MyBackendApp.Basics;

public static class ExtensionMethodDemo
{
    public static void Run()
    {
        string email = "  USER@EXAMPLE.COM  ".Trim();
        Console.WriteLine(email.HasBasicEmailShape()); // True

        string longText = "This is a very long product description that needs shortening.";
        Console.WriteLine(longText.Truncate(19)); // "This is a very long..."

        decimal price = 200.00m;
        decimal discountAmount = price.PercentOf(15m); // 30.00m
        Console.WriteLine(discountAmount);
    }
}
```

## 7. Backend Example: DI Registration and Composable Query Extensions

Extension methods are commonly used in ASP.NET Core to group related dependency-injection registrations. Query extensions can also package reusable `IQueryable<T>` transformations, provided they preserve the provider-backed query until its terminal operation.

### 7.1 Service Registration — `BillingServiceCollectionExtensions.cs`

```csharp
using System;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace MyBackendApp.Infrastructure.Extensions;

public static class BillingServiceCollectionExtensions
{
    public static IServiceCollection AddBillingInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        ArgumentNullException.ThrowIfNull(services);
        ArgumentNullException.ThrowIfNull(configuration);

        services.Configure<BillingOptions>(configuration.GetSection("Billing"));
        services.AddScoped<IPaymentGatewayService, StripePaymentGatewayService>();
        services.AddScoped<IInvoiceGenerator, PdfInvoiceGenerator>();

        return services;
    }

    public static IServiceCollection AddCachingInfrastructure(this IServiceCollection services)
    {
        ArgumentNullException.ThrowIfNull(services);
        services.AddMemoryCache();
        services.AddScoped<ICacheService, InMemoryCacheService>();
        return services;
    }
}

public sealed class BillingOptions
{
    public string ApiKey { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; } = 30;
}

// Minimal placeholders so the registration types are visible in this example.
public interface IPaymentGatewayService { }
public sealed class StripePaymentGatewayService : IPaymentGatewayService { }
public interface IInvoiceGenerator { }
public sealed class PdfInvoiceGenerator : IInvoiceGenerator { }
public interface ICacheService { }
public sealed class InMemoryCacheService : ICacheService { }
```

The configuration binding and caching extensions are supplied by the relevant `Microsoft.Extensions.*` packages (normally available through the ASP.NET Core shared framework). In `Program.cs`, the call can be composed with other service registrations, for example: `builder.Services.AddBillingInfrastructure(builder.Configuration).AddCachingInfrastructure();`.

### 7.2 Product Model — `Product.cs`

```csharp
using System;

namespace MyBackendApp.Core.Domain.Products;

public sealed class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Category { get; set; } = string.Empty;
    public bool IsActive { get; set; }
}
```

### 7.3 Query Helpers — `QueryableExtensions.cs`

```csharp
using System;
using System.Linq;

namespace MyBackendApp.Infrastructure.Persistence.Extensions;

public static class QueryableExtensions
{
    public static IQueryable<T> ApplyPaging<T>(
        this IQueryable<T> source,
        int pageNumber,
        int pageSize)
    {
        ArgumentNullException.ThrowIfNull(source);
        if (pageNumber < 1)
        {
            throw new ArgumentOutOfRangeException(nameof(pageNumber));
        }

        if (pageSize < 1)
        {
            throw new ArgumentOutOfRangeException(nameof(pageSize));
        }

        int offset = checked((pageNumber - 1) * pageSize);
        return source.Skip(offset).Take(pageSize);
    }

    public static IQueryable<T> ApplyIf<T>(
        this IQueryable<T> source,
        bool condition,
        Func<IQueryable<T>, IQueryable<T>> transformation)
    {
        ArgumentNullException.ThrowIfNull(source);
        ArgumentNullException.ThrowIfNull(transformation);
        return condition ? transformation(source) : source;
    }
}
```

These extensions return `IQueryable<T>` and don't call `ToList`, so `Skip`, `Take`, and any filters added by `transformation` remain part of the query definition. `ApplyIf` invokes the transformation while composing the query; because its input and result are `IQueryable<T>`, queryable operators such as `Where` add expression nodes for the provider to inspect. Callers should also enforce their API's maximum page size; positive values alone don't prevent a request for an impractically large page.

### 7.4 Composing and Executing the Query — `ProductQueryService.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using MyBackendApp.Core.Domain.Products;
using MyBackendApp.Infrastructure.Persistence.Extensions;

namespace MyBackendApp.Infrastructure.Persistence;

public sealed class ProductQueryService
{
    public Task<List<Product>> SearchProductsAsync(
        IQueryable<Product> products,
        string? categoryFilter,
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(products);

        string? category = string.IsNullOrWhiteSpace(categoryFilter)
            ? null
            : categoryFilter.Trim();

        IQueryable<Product> query = products
            .AsNoTracking()
            .Where(product => product.IsActive)
            .ApplyIf(
                category is not null,
                source => source.Where(product => product.Category == category))
            .OrderBy(product => product.Name)
            .ThenBy(product => product.Id)
            .ApplyPaging(pageNumber, pageSize);

        return query.ToListAsync(cancellationToken);
    }
}
```

Pass an EF Core query such as `dbContext.Products` while its `DbContext` is in scope. The helper methods compose filters and paging; `ToListAsync` is the execution point. Ordering by `Name` and then the unique `Id` makes page ordering deterministic. The provider's collation determines string comparison behavior for `Category` and `Name`, and large offset pages may call for keyset pagination instead.

## 8. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Extension method** | A static method callable with instance syntax through an extension receiver. | Adds reusable behavior without changing the extended type. |
| **Receiver / `this` parameter** | The first parameter in classic syntax; it identifies the extended type. | The caller supplies it implicitly as the method-call receiver. |
| **Extension block** | C# 14 syntax that groups extensions for a receiver type. | Can declare extension methods and other supported extension members. |
| **Extension resolution** | Compile-time lookup that prefers applicable real members before extensions. | Explains why extensions can't override a type's own behavior. |
| **Fluent API** | Chained calls where each operation returns a type usable by the next call. | Common for service registration and deferred query composition. |
| **Generic/interface extension** | An extension targeting an interface or open generic type. | Makes a helper available to many implementations or constructed types. |
| **Null-safe extension** | An extension whose receiver annotation and implementation intentionally handle `null`. | Useful for checks, but null safety must be implemented explicitly. |
| **Provider-preserving query extension** | An extension over `IQueryable<T>` that returns a composed query without materializing it. | Keeps filters and paging available to an EF Core provider for translation. |

## 9. Official References

- [Extension members — C# programming guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods) — classic extension methods, C# 14 extension blocks, scope, and member-resolution precedence.
- [How to implement and call a custom extension method — C#](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/how-to-implement-and-call-a-custom-extension-method) — syntax, namespace imports, and static-call equivalence.
- [The `extension` keyword — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/extension) — C# 14 extension block syntax.
- [`Enumerable` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable?view=net-10.0) — standard operators over `IEnumerable<T>`.
- [Dependency injection in ASP.NET Core — .NET 10](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0) — registering and composing services with `IServiceCollection`.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is LTS.
