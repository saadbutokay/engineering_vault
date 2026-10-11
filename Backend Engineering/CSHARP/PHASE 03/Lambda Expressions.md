Examples target **.NET 10 (LTS)** and **C# 14**, with nullable reference analysis enabled.

## 1. Conceptual Foundation

### 1.1 What Is a Lambda Expression?

A **lambda expression** is syntax for creating an anonymous function. The `=>` operator separates the input parameters from the body. Depending on the target type and the lambda's form, the compiler converts it to a delegate (such as `Func<T>` or `Action<T>`) or, for supported expression lambdas, builds an expression tree such as `Expression<Func<T, bool>>`.

Lambdas are often shorter than the older anonymous-method syntax (`delegate(int x) { return x * 2; }`) and are common in LINQ and callbacks. Anonymous methods remain valid C#; lambdas aren't a universal substitute for every anonymous-function scenario.

| Named method | Anonymous method | Lambda |
|---|---|---|
| `static int Double(int x) => x * 2;` | `delegate(int x) { return x * 2; }` | `x => x * 2` |

### 1.2 Target Typing and Common Forms

A lambda usually gets its parameter and return types from its **target delegate or expression-tree type**. The compiler often infers the parameter types, but it can't infer every lambda without context. For example, `Func<int, int> square = x => x * x;` supplies a target type. In modern C#, an explicitly typed lambda such as `var square = (int x) => x * x;` can also have a natural delegate type; `var square = x => x * x;` doesn't provide enough type information.

```csharp
using System;

public static class LambdaSyntaxDemo
{
    public static void Run()
    {
        // Expression lambda: the expression's result is returned.
        Func<int, int> square = x => x * x;

        // Statement lambda: a block can contain multiple statements.
        Func<int, long> factorial = n =>
        {
            if (n < 0)
            {
                throw new ArgumentOutOfRangeException(nameof(n));
            }

            long result = 1;
            for (int i = 2; i <= n; i++)
            {
                result = checked(result * i);
            }

            return result;
        };

        Func<int, int, int> add = (a, b) => a + b;
        Action greet = () => Console.WriteLine("Hello");
        Func<int, int> explicitlyTyped = (int x) => x * x;
        Func<int, string, int> ignoreSecond = (count, _) => count + 1;
        Func<int, int> nonCapturing = static x => x * 2;

        Console.WriteLine(square(5));
        Console.WriteLine(factorial(5));
        Console.WriteLine(add(3, 4));
        greet();
        Console.WriteLine(explicitlyTyped(6));
        Console.WriteLine(ignoreSecond(10, "unused"));
        Console.WriteLine(nonCapturing(7));
    }
}
```

The single `_` parameter above is simply unused; for backwards compatibility, a lone `_` in a lambda is treated as a parameter name. C# also supports discard parameters when multiple lambda parameters are unused. A `static` lambda can't capture local variables or instance state, which can make accidental captures easier to prevent.

## 2. Variable Capture and Closures

A lambda can refer to **outer variables** in scope where the lambda is defined. This is called **capturing**. The compiler preserves the captured variable's storage (often in a compiler-generated closure object), so the lambda observes the variable rather than a snapshot of its value. The captured variable remains alive while a delegate that uses it remains reachable.

### 2.1 Captured Variables Can Change

```csharp
using System;

public static class CaptureDemo
{
    public static void Run()
    {
        int multiplier = 2;
        Func<int, int> multiply = x => x * multiplier;

        Console.WriteLine(multiply(5)); // 10

        multiplier = 10;
        Console.WriteLine(multiply(5)); // 50: reads the updated captured variable
    }
}
```

### 2.2 The `for` Loop Capture Pitfall

Since C# 5, a `foreach` iteration variable is distinct for each iteration. A `for` loop's counter is still one variable shared by the loop, so deferred lambdas stored in a collection all observe its final value unless you introduce a per-iteration local.

```csharp
using System;
using System.Collections.Generic;

public static class LoopCaptureDemo
{
    public static void Run()
    {
        var actions = new List<Action>();

        for (int i = 0; i < 3; i++)
        {
            actions.Add(() => Console.WriteLine(i));
        }

        foreach (Action action in actions)
        {
            action();
        }
        // Prints 3, 3, 3: each lambda captures the same for-loop variable.

        var fixedActions = new List<Action>();
        for (int i = 0; i < 3; i++)
        {
            int captured = i; // A new local for this iteration.
            fixedActions.Add(() => Console.WriteLine(captured));
        }

        foreach (Action action in fixedActions)
        {
            action();
        }
        // Prints 0, 1, 2.
    }
}
```

## 3. Lambdas and Expression Trees

A lambda converted to a delegate, such as `Func<int, bool>`, is executable code. A supported **expression lambda** converted to `Expression<TDelegate>` becomes an inspectable, immutable expression-tree data structure. Call `.Compile()` to turn a tree into a delegate for local execution.

`IEnumerable<T>.Where` takes a delegate and evaluates it in-process. `IQueryable<T>.Where` takes an expression tree so its provider can inspect and potentially translate the query—for example, an EF Core provider may translate supported parts into SQL. A provider doesn't translate every .NET method or expression; unsupported expressions can fail translation or, in limited cases, be evaluated on the client. A statement lambda with a block body can't be converted to an expression tree.

```csharp
using System;
using System.Linq.Expressions;

public static class ExpressionTreeDemo
{
    public static void Run()
    {
        Func<int, bool> compiledDelegate = x => x > 18;
        Expression<Func<int, bool>> expressionTree = x => x > 18;

        bool result = compiledDelegate(21); // Executes the delegate: true.

        Func<int, bool> compiledFromTree = expressionTree.Compile();
        bool resultFromTree = compiledFromTree(21); // Also executes locally: true.

        Console.WriteLine($"Delegate result: {result}");
        Console.WriteLine($"Compiled tree result: {resultFromTree}");
        Console.WriteLine(expressionTree); // Displays the represented expression.
    }
}
```

An EF Core provider normally **consumes** the expression tree for translation; application code doesn't call `.Compile()` for a database filter. If a query contains an unsupported expression, behavior depends on the provider and query location—EF Core generally throws for untranslatable filters rather than silently loading all rows and running that filter locally.

## 4. Basic Syntax Example

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Basics;

public static class LambdaDemo
{
    public static void Run()
    {
        var numbers = new List<int> { 4, 8, 15, 16, 23, 42 };

        var evenNumbers = numbers.Where(n => n % 2 == 0).ToList();
        var doubled = numbers.Select(n => n * 2).ToList();
        int sum = numbers.Aggregate(0, (accumulator, n) => accumulator + n);

        Console.WriteLine(string.Join(", ", evenNumbers));
        Console.WriteLine(string.Join(", ", doubled));
        Console.WriteLine(sum);

        Func<int, bool> isPrime = n =>
        {
            if (n < 2)
            {
                return false;
            }

            for (int divisor = 2; divisor <= n / divisor; divisor++)
            {
                if (n % divisor == 0)
                {
                    return false;
                }
            }

            return true;
        };

        var primes = numbers.Where(isPrime).ToList();
        Console.WriteLine(string.Join(", ", primes));

        int threshold = 10;
        Func<int, bool> aboveThreshold = n => n > threshold;
        Console.WriteLine(numbers.Count(aboveThreshold));
    }
}
```

The LINQ methods above operate on an in-memory `List<int>`, so their lambdas are used as delegates. The threshold predicate captures a local variable.

## 5. Backend Example: Composable Product Specifications

A reusable **specification** can store a predicate as `Expression<Func<T, bool>>`. The same criteria can then be compiled for in-memory filtering or passed to an `IQueryable<T>` provider, which may translate supported expression nodes. The provider's translation support still matters; the expression tree doesn't guarantee that every C# operation becomes SQL.

The code is split into separate files because a single C# source file can't contain multiple file-scoped namespace declarations (`namespace Name;`).

### 5.1 `Specification<T>.cs`

```csharp
#nullable enable
using System;
using System.Linq.Expressions;

namespace MyBackendApp.Core.Specifications;

public sealed class Specification<T>
{
    public Expression<Func<T, bool>> Criteria { get; }

    public Specification(Expression<Func<T, bool>> criteria)
    {
        ArgumentNullException.ThrowIfNull(criteria);
        Criteria = criteria;
    }

    public Specification<T> And(Specification<T> other)
    {
        ArgumentNullException.ThrowIfNull(other);

        var parameter = Expression.Parameter(typeof(T), "entity");
        Expression left = new ReplaceParameterVisitor(Criteria.Parameters[0], parameter)
            .Visit(Criteria.Body)!;
        Expression right = new ReplaceParameterVisitor(other.Criteria.Parameters[0], parameter)
            .Visit(other.Criteria.Body)!;

        BinaryExpression body = Expression.AndAlso(left, right);
        return new Specification<T>(Expression.Lambda<Func<T, bool>>(body, parameter));
    }

    public Func<T, bool> ToPredicate() => Criteria.Compile();

    private sealed class ReplaceParameterVisitor : ExpressionVisitor
    {
        private readonly ParameterExpression _oldParameter;
        private readonly ParameterExpression _newParameter;

        public ReplaceParameterVisitor(
            ParameterExpression oldParameter,
            ParameterExpression newParameter)
        {
            _oldParameter = oldParameter;
            _newParameter = newParameter;
        }

        protected override Expression VisitParameter(ParameterExpression node)
        {
            return node == _oldParameter ? _newParameter : base.VisitParameter(node)!;
        }
    }
}
```

### 5.2 Product Model — `Product.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Domain.Products;

public sealed record Product(
    Guid Id,
    string Name,
    decimal Price,
    int StockQuantity,
    string Category);
```

### 5.3 Product Specifications — `ProductSpecifications.cs`

```csharp
#nullable enable
using System;
using MyBackendApp.Core.Domain.Products;
using MyBackendApp.Core.Specifications;

namespace MyBackendApp.Core.Application.Products;

public static class ProductSpecifications
{
    public static Specification<Product> InStock() =>
        new(product => product.StockQuantity > 0);

    public static Specification<Product> PriceBelow(decimal maxPrice) =>
        new(product => product.Price < maxPrice);

    public static Specification<Product> InCategory(string category)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(category);
        return new Specification<Product>(product => product.Category == category);
    }
}
```

The `==` string comparison is straightforward to translate, but case-sensitivity depends on the database collation. In-memory string comparison and a database's collation may not behave identically. For consistent case-insensitive matching, choose an explicit normalization or collation strategy supported by the target provider.

### 5.4 Apply Specifications — `ProductSearchService.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;
using MyBackendApp.Core.Domain.Products;
using MyBackendApp.Core.Specifications;

namespace MyBackendApp.Core.Application.Products;

public sealed class ProductSearchService
{
    public IQueryable<Product> SearchQueryable(
        IQueryable<Product> products,
        decimal maxPrice)
    {
        ArgumentNullException.ThrowIfNull(products);
        Specification<Product> specification = BuildSpecification(maxPrice);

        // Keeps the expression tree for the query provider to inspect.
        return products.Where(specification.Criteria);
    }

    public IReadOnlyList<Product> SearchInMemory(
        IEnumerable<Product> products,
        decimal maxPrice)
    {
        ArgumentNullException.ThrowIfNull(products);
        Func<Product, bool> predicate = BuildSpecification(maxPrice).ToPredicate();
        return products.Where(predicate).ToList();
    }

    private static Specification<Product> BuildSpecification(decimal maxPrice) =>
        ProductSpecifications.InStock()
            .And(ProductSpecifications.PriceBelow(maxPrice))
            .And(ProductSpecifications.InCategory("Electronics"));
}
```

`SearchQueryable` returns a deferred query; a database-backed caller can pass an EF Core `DbSet<Product>` and enumerate/execute the query later. `SearchInMemory` explicitly compiles the expression and applies a delegate locally. The two paths are separate because compiling first would prevent the query provider from seeing the predicate tree.

## 6. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Lambda expression** | Anonymous-function syntax using `=>`. | Concisely supplies behavior to callbacks and LINQ operators. |
| **Expression lambda** | A lambda whose body is a single expression. | Can target a delegate or, when supported, an expression tree. |
| **Statement lambda** | A lambda whose body is a block of statements. | Useful for multi-step delegate logic; can't be converted to an expression tree. |
| **Target typing** | Using the expected delegate or expression-tree type to infer lambda parameter/result types. | Enables concise LINQ predicates and transformations. |
| **Capture** | Referring to variables from an enclosing scope. | Supports parameterized predicates, but captured state can outlive the method scope. |
| **Closure** | The lambda together with the preserved state of captured variables. | Explains updated values and object-lifetime behavior. |
| **Expression tree** | An inspectable data structure representing supported code expressions. | Lets `IQueryable` providers translate or inspect predicates. |
| **Specification pattern** | Encapsulating reusable, composable business criteria. | Can provide both local delegates and provider-visible expression trees, subject to translation support. |

## 7. Official References

- [Lambda expressions and anonymous functions — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions) — lambda forms, inference, captures, static lambdas, and LINQ examples.
- [Expression Trees — C#](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/) — tree conversion, inspection, compilation, and language limitations.
- [`Expression<TDelegate>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression-1?view=net-10.0) — strongly typed lambda expression tree API.
- [Client vs. Server Evaluation — EF Core](https://learn.microsoft.com/en-us/ef/core/querying/client-eval) — provider translation boundaries and client-evaluation behavior.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is LTS.
