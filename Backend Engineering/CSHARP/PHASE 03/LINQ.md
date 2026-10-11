Examples target **.NET 10 (LTS)** and **C# 14**. The backend examples assume an EF Core relational provider; actual SQL translation depends on the provider and its version.

## 1. Conceptual Foundation

### 1.1 What Is LINQ?

**LINQ** (Language Integrated Query) combines C# query syntax with standard query-operator extension methods, primarily in `System.Linq`. It provides a consistent way to filter, project, order, group, join, and aggregate data from sources such as in-memory collections, database providers, and XML.

LINQ is commonly described as **declarative**: describe the result you want rather than writing every loop and accumulator yourself. The standard operators are implemented primarily by `Enumerable` for `IEnumerable<T>` sources and `Queryable` for `IQueryable<T>` sources.

The following comparison assumes `employees` is a collection of `Employee` objects:

```csharp
#nullable enable
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Examples.ImperativeVsLinq;

public sealed record Employee(string Name, decimal Salary);

public static class ImperativeVsLinqDemo
{
    public static List<string> WithLoops(IEnumerable<Employee> employees)
    {
        var highEarners = new List<Employee>();
        foreach (Employee employee in employees)
        {
            if (employee.Salary > 100_000m)
            {
                highEarners.Add(employee);
            }
        }

        highEarners.Sort((left, right) => right.Salary.CompareTo(left.Salary));
        var names = new List<string>();
        foreach (Employee employee in highEarners)
        {
            names.Add(employee.Name);
        }

        return names;
    }

    public static List<string> WithLinq(IEnumerable<Employee> employees) =>
        employees
            .Where(employee => employee.Salary > 100_000m)
            .OrderByDescending(employee => employee.Salary)
            .Select(employee => employee.Name)
            .ToList();
}
```

### 1.2 Query Syntax and Method Syntax

The two common LINQ styles are **query syntax** (SQL-like) and **method syntax** (chained operator calls). Query syntax is translated by the C# compiler into calls that follow the LINQ query pattern; many queries can be written either way. Some operations—such as `Count`, `Any`, `First`, and `ToList`—are available only as method calls.

```csharp
using System.Linq;

string[] names = ["Ada", "Grace", "Linus"];

// Method syntax
var methodQuery = names
    .Where(name => name.Length > 4)
    .OrderBy(name => name)
    .Select(name => name.ToUpperInvariant());

// Query syntax: equivalent result and deferred behavior
var querySyntax =
    from name in names
    where name.Length > 4
    orderby name
    select name.ToUpperInvariant();
```

Pick the form that makes the query easiest to understand. Query syntax can be especially readable for joins and grouping; method syntax is often convenient for incremental composition and terminal operators. Both still depend on the source type and its LINQ provider.

## 2. `IEnumerable<T>` and `IQueryable<T>`

| Feature | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Typical operators | `Enumerable.Where`, `Enumerable.Select` | `Queryable.Where`, `Queryable.Select` |
| Lambda representation | Usually a delegate such as `Func<T, bool>` | Usually an expression tree such as `Expression<Func<T, bool>>` |
| Where the predicate runs | In the .NET process as the sequence is enumerated | A provider inspects the expression and may translate supported parts to another query language, such as SQL |
| Typical use | Collections already in memory, files or custom iterators | EF Core `DbSet<T>` and other query-provider-backed sources |

`IQueryable<T>` also implements `IEnumerable<T>`, but the **compile-time type** usually determines which LINQ extension-method overload is selected. If an `IQueryable<T>` is converted to `IEnumerable<T>` or `AsEnumerable()` is called, later operators normally use `Enumerable` and run in .NET. `AsEnumerable()` changes the operator boundary; it doesn't itself materialize the rows. `ToList()` does materialize.

```csharp
#nullable enable
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

namespace MyBackendApp.Basics;

public static class QueryExecutionDemo
{
    // Requires an EF Core-backed source. Employee is the model from this guide.
    public static async Task<(List<Employee> ServerFiltered, List<Employee> ClientFiltered)>
        CompareAsync(IQueryable<Employee> source, CancellationToken cancellationToken)
    {
        // Queryable.Where adds an expression to the provider's query definition.
        List<Employee> serverFiltered = await source
            .Where(employee => employee.Salary > 100_000m)
            .ToListAsync(cancellationToken);

        // Demonstration only: this materializes every row before filtering locally.
        IEnumerable<Employee> allRows = await source.ToListAsync(cancellationToken);
        List<Employee> clientFiltered = allRows
            .Where(employee => employee.Salary > 100_000m)
            .ToList();

        return (serverFiltered, clientFiltered);
    }
}
```

The first pipeline can filter in the database before rows are returned, if the provider translates the predicate. The second fetches the complete source first and applies `Enumerable.Where` locally. Avoid loading all rows this way for a large table. An `IEnumerable<T>` source isn't inherently materialized or eager: it may be lazy or streaming; the example becomes fully loaded specifically because it calls `ToListAsync()` first.

## 3. Query Construction, Execution, and Buffering

Creating a query usually creates a **query definition**, not a set of results. For LINQ to Objects, sequence-returning operators such as `Where` and `Select` are commonly deferred until enumeration. Scalar-returning operators such as `Count`, `Any`, `First`, `Sum`, and `Average` execute immediately. `ToList` and `ToArray` also force execution and store the results.

For `IQueryable<T>`, query-building calls normally extend an expression tree. A provider such as EF Core translates and executes the query when it is enumerated or when an execution operator such as `ToListAsync`, `CountAsync`, or `AnyAsync` is awaited. Simple filter, ordering, projection, and paging operations are often sent as one database query, but LINQ syntax alone does not guarantee a particular SQL shape, number of commands, or optimization.

> **Deferred does not always mean streaming.** With LINQ to Objects, `Where` and `Select` can usually yield elements as they are read. `OrderBy` and `GroupBy` are deferred, but generally need to consume and buffer the source before they can produce their output. A deferred query may also run again every time it is enumerated.

A query's values are generally determined when it executes, not when the query variable is created. If the source changes before execution—or between repeated enumerations—the results can change.

## 4. Core LINQ Operator Categories

| Category | Representative methods | Purpose |
|---|---|---|
| Filtering | `Where`, `OfType<T>` | Keep elements that match a condition or type. |
| Projection | `Select`, `SelectMany` | Transform each element; `SelectMany` flattens nested sequences. |
| Ordering | `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending` | Sort by one or more keys. Use `ThenBy` to add a secondary key without replacing the earlier ordering. |
| Grouping | `GroupBy` | Group elements by a key, producing `IGrouping<TKey, TElement>` values in LINQ to Objects. |
| Joining | `Join`, `GroupJoin` | Combine sequences by matching keys; provider translation varies, especially for grouped results. |
| Aggregation | `Sum`, `Count`, `LongCount`, `Average`, `Min`, `Max`, `Aggregate` | Reduce a sequence to a scalar result. |
| Quantifiers | `Any`, `All`, `Contains` | Test whether a match, universal condition, or value exists. |
| Element access | `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `ElementAt` | Retrieve an element, with either exception or default behavior. |
| Set operations | `Distinct`, `Union`, `Intersect`, `Except` | Apply set-like operations using the applicable equality rules. |
| Partitioning | `Skip`, `Take`, `SkipWhile`, `TakeWhile` | Select a portion of a sequence; often used for paging. |
| Materialization | `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup` | Execute and store results in a collection. `ToDictionary` throws if keys aren't unique. |

The standard operators are not all translated by every `IQueryable` provider. Even EF Core relational providers can differ in which query shapes or methods they translate. In particular, grouping into aggregate values is a common SQL pattern; returning arbitrary `IGrouping` objects is not itself a natural relational result and may be evaluated differently. Verify complex query translation against the provider you deploy.

## 5. Important Behavioral Nuances

### 5.1 `First` and `Single` Variants

| Operator | Empty result | More than one match | Appropriate when |
|---|---|---|---|
| `First()` | Throws `InvalidOperationException` | Returns the first match | At least one match is expected; multiple matches are acceptable. |
| `FirstOrDefault()` | Returns `default(T)` | Returns the first match | No match is acceptable. |
| `Single()` | Throws `InvalidOperationException` | Throws `InvalidOperationException` | Exactly one match is a business invariant, such as a unique identifier lookup. |
| `SingleOrDefault()` | Returns `default(T)` | Throws `InvalidOperationException` | Zero or one match is acceptable, but duplicates indicate a problem. |

For reference types, `default(T)` is usually `null`; for value types it may be a value such as `0`, which can be ambiguous. .NET also provides overloads of `FirstOrDefault` and related operators that accept an explicit default value. If the query is provider-backed, check that provider's translation support for the overload you choose. Without an explicit `OrderBy`, a database query's “first” row is not guaranteed to be a particular row.

### 5.2 Deferred Queries and Multiple Enumeration

Enumerating the same deferred `IEnumerable<T>` query more than once reruns its operators. For an EF Core `IQueryable<T>`, each terminal operation generally sends a separate database query.

```csharp
#nullable enable
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Basics;

public static class MultipleEnumerationDemo
{
    public static (int Count, List<Employee> Items) EnumerateTwice(
        IEnumerable<Employee> employees)
    {
        IEnumerable<Employee> query = employees
            .Where(employee => employee.Salary > 100_000m);

        int count = query.Count(); // Enumerates the pipeline.
        List<Employee> list = query.ToList(); // Enumerates it again.
        return (count, list);
    }

    public static (int Count, List<Employee> Items) MaterializeOnce(
        IEnumerable<Employee> employees)
    {
        List<Employee> snapshot = employees
            .Where(employee => employee.Salary > 100_000m)
            .ToList();

        return (snapshot.Count, snapshot);
    }
}
```

For a database query, `CountAsync()` followed by `ToListAsync()` normally means two database commands. That can be intentional—for example, when a UI needs both a total count and a page—but if only the count is needed, issue only `CountAsync()`. Don't materialize a large, unbounded query solely to avoid a second enumeration; use the operation that matches the required result and resource limits.

## 6. Basic Syntax Example

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Basics;

public sealed record Employee(string Name, string Department, decimal Salary);

public static class LinqDemo
{
    public static void Run()
    {
        List<Employee> employees =
        [
            new("Alice", "Engineering", 120_000m),
            new("Bob", "Engineering", 95_000m),
            new("Carla", "Sales", 85_000m),
            new("David", "Sales", 105_000m),
            new("Elena", "Finance", 130_000m)
        ];

        // Filtering, ordering, projection, and materialization.
        List<string> highEarnerNames = employees
            .Where(employee => employee.Salary > 100_000m)
            .OrderByDescending(employee => employee.Salary)
            .Select(employee => employee.Name)
            .ToList();

        Console.WriteLine(string.Join(", ", highEarnerNames));

        // Grouping and aggregation. The sequence is evaluated by the foreach.
        var departmentSummaries = employees
            .GroupBy(employee => employee.Department)
            .Select(group => new
            {
                Department = group.Key,
                AverageSalary = group.Average(employee => employee.Salary),
                Count = group.Count()
            })
            .OrderByDescending(summary => summary.AverageSalary);

        foreach (var summary in departmentSummaries)
        {
            Console.WriteLine(
                $"{summary.Department}: Avg={summary.AverageSalary:C}, Count={summary.Count}");
        }

        // Quantifiers and element access.
        bool anyAboveThreshold = employees.Any(employee => employee.Salary > 125_000m);
        Employee richest = employees.OrderByDescending(employee => employee.Salary).First();

        Console.WriteLine($"Any above 125k: {anyAboveThreshold}, Richest: {richest.Name}");
    }
}
```

This example uses an in-memory `List<Employee>`, so `Enumerable` operators run in the application. Its `GroupBy` result is deferred until enumeration, but LINQ to Objects generally buffers the source to form groups.

## 7. Backend Example: EF Core Sales Reporting

The following example demonstrates filtering, projection, grouping, joining, aggregation, pagination, and asynchronous execution with an EF Core context. The code is split into separate `.cs` files; each uses a single file-scoped namespace. In a real project, configure and register the context with the database provider you use.

### 7.1 Entities — `Customer.cs` and `Order.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Domain.Sales;

public sealed class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Region { get; set; } = string.Empty;
}
```

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Domain.Sales;

public sealed class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public decimal Amount { get; set; }
    public DateTimeOffset PlacedAtUtc { get; set; }
    public string Status { get; set; } = string.Empty;
}
```

### 7.2 Context — `SalesDbContext.cs`

```csharp
#nullable enable
using Microsoft.EntityFrameworkCore;
using MyBackendApp.Core.Domain.Sales;

namespace MyBackendApp.Core.Infrastructure.Persistence;

public sealed class SalesDbContext : DbContext
{
    public SalesDbContext(DbContextOptions<SalesDbContext> options)
        : base(options)
    {
    }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
}
```

### 7.3 Reporting Service — `SalesReportingService.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using MyBackendApp.Core.Infrastructure.Persistence;

namespace MyBackendApp.Core.Application.Reporting;

public sealed record RegionalSalesSummary(
    string Region,
    decimal TotalRevenue,
    int OrderCount,
    decimal AverageOrderValue);

public sealed record CustomerOrderView(
    Guid OrderId,
    string CustomerName,
    decimal OrderAmount,
    DateTimeOffset PlacedAtUtc);

public sealed class SalesReportingService
{
    private readonly SalesDbContext _dbContext;

    public SalesReportingService(SalesDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<List<RegionalSalesSummary>> GetRegionalSalesSummaryAsync(
        DateTimeOffset since,
        CancellationToken cancellationToken = default)
    {
        var query =
            from order in _dbContext.Orders.AsNoTracking()
            where order.Status == "Completed" && order.PlacedAtUtc >= since
            join customer in _dbContext.Customers.AsNoTracking()
                on order.CustomerId equals customer.Id
            group order by customer.Region into regionGroup
            select new RegionalSalesSummary(
                regionGroup.Key,
                regionGroup.Sum(order => order.Amount),
                regionGroup.Count(),
                regionGroup.Average(order => order.Amount));

        return await query
            .OrderByDescending(summary => summary.TotalRevenue)
            .ToListAsync(cancellationToken);
    }

    public async Task<List<CustomerOrderView>> GetRecentOrdersPageAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        if (pageNumber < 1)
        {
            throw new ArgumentOutOfRangeException(nameof(pageNumber));
        }

        if (pageSize < 1)
        {
            throw new ArgumentOutOfRangeException(nameof(pageSize));
        }

        int offset = checked((pageNumber - 1) * pageSize);

        var query =
            from order in _dbContext.Orders.AsNoTracking()
            where order.Status != "Cancelled"
            join customer in _dbContext.Customers.AsNoTracking()
                on order.CustomerId equals customer.Id
            orderby order.PlacedAtUtc descending, order.Id
            select new CustomerOrderView(
                order.Id,
                customer.Name,
                order.Amount,
                order.PlacedAtUtc);

        // Ordering by timestamp plus unique OrderId makes paging deterministic.
        return await query
            .Skip(offset)
            .Take(pageSize)
            .ToListAsync(cancellationToken);
    }

    public Task<bool> CustomerHasOutstandingOrdersAsync(
        Guid customerId,
        CancellationToken cancellationToken = default) =>
        _dbContext.Orders.AnyAsync(
            order => order.CustomerId == customerId && order.Status == "Pending",
            cancellationToken);
}
```

`Where`, `OrderBy`, `Join`, and similar calls build the query; they don't have EF Core `Async` suffixes because they don't perform database I/O themselves. `ToListAsync` and `AnyAsync` execute the query asynchronously. .NET 10 also provides LINQ operators for `IAsyncEnumerable<T>`; those operate on asynchronous streams and are distinct from EF Core's `IQueryable<T>` expression-tree composition.

The service's filters, join, scalar grouping aggregates, and pagination are common relational query shapes, but translation is provider-dependent: inspect generated SQL and test with the database provider used in production. A successful C# compile does not prove a provider can translate a query.

The page query orders by both timestamp and unique order ID before `Skip`/`Take`. Pagination should use a fully unique ordering. Offset paging can also become expensive for very large offsets; keyset (seek-based) pagination may be a better fit when users move one page at a time rather than jump to arbitrary page numbers.

For read-heavy queries, project only the columns needed into a DTO and filter/page before materializing. EF Core's async extensions are in `Microsoft.EntityFrameworkCore`; an EF Core `DbContext` also shouldn't run multiple parallel operations at once—await each operation before starting another on the same context.

## 8. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **LINQ** | C# query syntax plus standard query operators for working with data sources. | Provides a shared vocabulary for filtering, shaping, and aggregating data. |
| **Query syntax** | SQL-like C# query expression that the compiler translates to query-pattern method calls. | Often readable for joins and grouping. |
| **Method syntax** | Chained calls such as `.Where(...).Select(...)`. | Supports all standard operators and works well for incremental composition. |
| **`IEnumerable<T>`** | A sequence enumerated by .NET; LINQ-to-Objects operators typically use delegates. | Appropriate for already-loaded data or custom local iterators. |
| **`IQueryable<T>`** | A provider-backed query abstraction that exposes an expression tree. | Enables a provider such as EF Core to translate supported operations. |
| **Deferred execution** | Building a query now and evaluating it when enumerated or executed. | Enables composition, but repeated enumeration can repeat work or database calls. |
| **Materialization** | Executing a query and storing results in a collection such as `List<T>`. | Makes results reusable in memory, with memory and freshness tradeoffs. |
| **Terminal operator** | An operator that returns a scalar or materializes a sequence, triggering execution. | Defines when a query runs and often when a database round-trip occurs. |
| **Client evaluation** | Running part of a query in .NET instead of on the database. | Must be deliberate to avoid transferring and processing excessive data. |

## 9. Official References

- [Introduction to LINQ Queries — C#](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries) — query construction, execution, deferred and immediate operations, streaming, and buffering.
- [Write LINQ queries — C#](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/write-linq-queries) — query syntax, method syntax, extension methods, and composability.
- [Standard query operators overview — C#](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/) — operator categories and `IEnumerable<T>` versus `IQueryable<T>`.
- [Client vs. Server Evaluation — EF Core](https://learn.microsoft.com/en-us/ef/core/querying/client-eval) — client evaluation rules and explicit client-side boundaries.
- [Complex Query Operators — EF Core](https://learn.microsoft.com/en-us/ef/core/querying/complex-query-operators) — translation behavior for joins, grouping, and aggregates.
- [Efficient Querying — EF Core](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying) — projection, limiting results, and query efficiency.
- [Pagination — EF Core](https://learn.microsoft.com/en-us/ef/core/querying/pagination) — unique ordering, offset paging, and keyset pagination.
- [Asynchronous Programming — EF Core](https://learn.microsoft.com/en-us/ef/core/miscellaneous/async) — async execution methods and `IAsyncEnumerable<T>` notes for .NET 10.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is LTS.
