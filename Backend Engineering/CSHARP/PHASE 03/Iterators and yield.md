Examples target **.NET 10 (LTS)** with **C# 14**. Synchronous iterators use `IEnumerable<T>`/`IEnumerator<T>`; asynchronous iterators use `IAsyncEnumerable<T>`/`IAsyncEnumerator<T>` and are introduced here before their fuller treatment in asynchronous programming.

## 1. Conceptual Foundation

### 1.1 What Is an Iterator?

An **iterator** provides sequential access to values, commonly through `foreach`. A C# iterator block uses `yield return` to provide values as the consumer asks for them, rather than first building and returning a complete collection. Iterator blocks are commonly methods; they can also appear in a property or indexer's `get` accessor when it returns a supported iterator type.

| Eager approach | Iterator approach |
|---|---|
| Builds a collection such as `List<int>` before returning. | Computes or fetches the next value as enumeration advances. |
| Can retain the entire result in memory. | Can keep memory use low when the source and consumer also process incrementally. |
| Work is generally done before the caller can use the first result. | Work is deferred until enumeration and can stop early. |

The compiler generates the iterator's state-machine machinery. It preserves the current position and local state across `yield return` suspension points, and implements the appropriate enumeration interfaces. This is conceptually related to the state machines generated for `async`/`await`, but ordinary iterators are **synchronous, pull-based** sequences.

### 1.2 Why Iterators Matter in Backend Engineering

Iterators can help with:

- Processing file contents or large result sets incrementally.
- Composing lazy LINQ-style pipelines.
- Traversing trees, organizational hierarchies, or other domain structures.
- Reading paged data without requesting every page in advance.

**Important:** `yield` alone doesn't guarantee constant memory or database streaming. The source may already be a materialized list, a provider may buffer results, a consumer may call `ToList()`, or an iterator may hold an entire page at a time. A streaming design depends on the whole pipeline, including source, provider, consumer, and resource lifetime.

## 2. The `yield` Keyword

### 2.1 `yield return`

`yield return value;` provides the next element and suspends the iterator. The consumer processes that value; on the next `MoveNext()` (or `MoveNextAsync()`), execution resumes after the `yield return` with local variables preserved.

### 2.2 `yield break`

`yield break;` ends the current iteration immediately. Reaching the end of the iterator body also ends the sequence. An iterator can't use `return someCollection;` to return a sequence; use `yield return` for elements and `yield break` for early termination.

### 2.3 Deferred Execution

Calling a synchronous iterator method creates an enumerable, but the iterator body starts when enumeration advances—typically at the first `MoveNext()` inside `foreach`. A sequence can be enumerated more than once; each enumeration normally starts the iterator again, so side effects or I/O may happen again.

```csharp
using System;
using System.Collections.Generic;

public static class LoggingIteratorDemo
{
    public static IEnumerable<int> GetNumbersWithLogging()
    {
        Console.WriteLine("Iterator started.");

        for (int i = 0; i < 3; i++)
        {
            Console.WriteLine($"About to yield {i}");
            yield return i;
        }
    }

    public static void Run()
    {
        IEnumerable<int> sequence = GetNumbersWithLogging();
        Console.WriteLine("Sequence created; iterator body hasn't run yet.");

        foreach (int number in sequence)
        {
            Console.WriteLine($"Received: {number}");
        }
    }
}
```

The call to `GetNumbersWithLogging()` doesn't print `Iterator started.`. Enumeration does. Calling `.ToList()` also starts enumeration, but consumes the whole sequence and materializes its values.

If a parameter must be validated **at method-call time**, put the checks in a non-iterator wrapper and return a separate local or private iterator. Code inside the iterator body—including code before its first `yield return`—runs only during enumeration.

### 2.4 State, Disposal, and Resource Lifetime

An iterator may keep a file reader or other disposable resource alive across yields. When enumeration completes—or when the enumerator is disposed after an early exit—the iterator's `finally`/`using` cleanup runs. `foreach` disposes its enumerator, including when the loop exits with `break`.

This is useful for resource-backed streaming, but it means the resource remains open while the caller processes each item. Keep enumeration lifetimes short and dispose enumerators correctly. Avoid returning an iterator that depends on a resource that the caller has already disposed.

### 2.5 Iterator Restrictions and Practical Notes

- Iterator methods can't have `ref`, `in`, or `out` parameters, and `yield` statements can't appear in lambda expressions or anonymous methods.
- `yield return` and `yield break` can't appear in `catch` or `finally` blocks, or in a `try` block that has a `catch`. A `try` with `finally` and no `catch` is allowed, which is how `using` cleanup works.
- Exceptions in the iterator body are usually observed when enumeration reaches the throwing code, not when the iterator method is called.
- Use synchronous iterators for synchronous work. For network or other asynchronous I/O, use an async iterator and `await foreach` rather than blocking inside `MoveNext()`.

## 3. Iterator Return Types

| Iterator kind | Return type | Typical consumer |
|---|---|---|
| Synchronous sequence | `IEnumerable<T>` or non-generic `IEnumerable` | `foreach` |
| Synchronous cursor | `IEnumerator<T>` or non-generic `IEnumerator` | Usually a custom `GetEnumerator()` implementation; this represents an active enumerator rather than a reusable sequence. |
| Asynchronous sequence | `IAsyncEnumerable<T>` | `await foreach` |
| Asynchronous cursor | `IAsyncEnumerator<T>` | Usually a custom `GetAsyncEnumerator()` implementation or explicit cursor management. |

For an async iterator, declare `async` and use `IAsyncEnumerable<T>` or `IAsyncEnumerator<T>`; its body can use both `await` and `yield return`. A synchronous `IEnumerable<T>` iterator performs synchronous work even if the source is an API or database.

## 4. Basic Syntax Example

This example generates even numbers as `long` values so the sequence doesn't overflow `int` for large positive `maxCount`. The outer `GetLinesUntilMarker` method validates its arguments immediately, then returns a separate lazy iterator.

```csharp
#nullable enable
using System;
using System.Collections.Generic;

namespace MyBackendApp.Basics;

public static class IteratorDemo
{
    public static IEnumerable<long> GenerateEvenNumbers(int maxCount)
    {
        if (maxCount <= 0)
        {
            yield break;
        }

        long current = 0;
        for (int count = 0; count < maxCount; count++)
        {
            yield return current;
            current += 2;
        }
    }

    public static IEnumerable<string> GetLinesUntilMarker(
        IEnumerable<string> lines,
        string stopMarker)
    {
        ArgumentNullException.ThrowIfNull(lines);
        ArgumentNullException.ThrowIfNull(stopMarker);

        return Iterate(lines, stopMarker);

        static IEnumerable<string> Iterate(IEnumerable<string> source, string marker)
        {
            foreach (string line in source)
            {
                if (line == marker)
                {
                    yield break;
                }

                yield return line;
            }
        }
    }

    public static void Run()
    {
        foreach (long number in GenerateEvenNumbers(5))
        {
            Console.WriteLine(number); // 0, 2, 4, 6, 8
        }

        string[] lines = ["header", "row-1", "--END--", "row-2"];
        foreach (string line in GetLinesUntilMarker(lines, "--END--"))
        {
            Console.WriteLine(line); // header, row-1
        }
    }
}
```

Collection expressions such as `[...]` require C# 12 or later; the .NET 10 SDK defaults to C# 14 unless `LangVersion` is overridden.

## 5. Backend Examples: Incremental Export and Paged API Consumption

The examples are separated into files. This matters because a C# file can contain only one file-scoped namespace declaration (`namespace Name;`); combining several such declarations in one source file is invalid.

### 5.1 Synthetic Record Source — `TransactionDataSource.cs`

This generator demonstrates deferred record creation; it is **not** a database query or a simulation of database I/O.

```csharp
#nullable enable
using System;
using System.Collections.Generic;

namespace MyBackendApp.Infrastructure.Export;

public sealed record TransactionRecord(Guid Id, decimal Amount, DateTimeOffset OccurredAtUtc);

public sealed class TransactionDataSource
{
    private readonly int _totalRecordCount;

    public TransactionDataSource(int totalRecordCount)
    {
        if (totalRecordCount < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(totalRecordCount));
        }

        _totalRecordCount = totalRecordCount;
    }

    public IEnumerable<TransactionRecord> StreamAllTransactions()
    {
        for (int i = 0; i < _totalRecordCount; i++)
        {
            yield return new TransactionRecord(
                Id: Guid.NewGuid(),
                Amount: 10.00m + i,
                OccurredAtUtc: DateTimeOffset.UnixEpoch.AddMinutes(i));
        }
    }
}
```

Records are created only as the consumer advances. Because IDs are newly generated each time, enumerating this synthetic sequence twice produces different records; don't mistake a deferred sequence for a cached snapshot.

### 5.2 CSV Export — `CsvExportService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;

namespace MyBackendApp.Infrastructure.Export;

public static class CsvExportService
{
    public static void ExportToCsv(IEnumerable<TransactionRecord> transactions, string filePath)
    {
        ArgumentNullException.ThrowIfNull(transactions);
        ArgumentNullException.ThrowIfNull(filePath);

        using var writer = new StreamWriter(filePath);
        writer.WriteLine("Id,Amount,OccurredAtUtc");

        foreach (TransactionRecord transaction in transactions)
        {
            string amount = transaction.Amount.ToString(CultureInfo.InvariantCulture);
            string occurredAt = transaction.OccurredAtUtc.ToString("O", CultureInfo.InvariantCulture);
            writer.WriteLine($"{transaction.Id:D},{amount},{occurredAt}");
        }
    }
}
```

The exporter processes one record at a time and doesn't materialize the input itself. That memory benefit depends on passing it a genuinely incremental source; passing a pre-built `List<TransactionRecord>` still means the records are already in memory. The shown fields are formatted invariantly and contain no commas; if exporting arbitrary text fields, use correct CSV quoting/escaping. For example, pass `new TransactionDataSource(1_000_000).StreamAllTransactions()` directly to `ExportToCsv` rather than inserting a `.ToList()` step.

For real database exports, verify the ORM/provider's buffering and streaming behavior, and ensure the database reader/context remains alive for the entire enumeration. For cancellable file or database I/O, use the relevant asynchronous APIs.

### 5.3 Partner API Contract — `IPartnerApi.cs`

A page-based API usually returns one bounded page at a time. The API implementation is omitted; it would use an HTTP client and honor cancellation.

```csharp
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.ExternalApi;

public sealed record ExternalApiPage<T>(IReadOnlyList<T> Items, bool HasNextPage);

public interface IPartnerApi
{
    Task<ExternalApiPage<string>> FetchPageAsync(
        int pageNumber,
        CancellationToken cancellationToken);
}
```

### 5.4 Lazy Async Page Iterator — `PartnerApiClient.cs`

The next page is requested only when enumeration reaches it. This models asynchronous I/O with `IAsyncEnumerable<T>`; the actual HTTP client should also apply timeouts, retries, and the partner's rate limits.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.ExternalApi;

public sealed class PartnerApiClient
{
    private readonly IPartnerApi _api;

    public PartnerApiClient(IPartnerApi api)
    {
        ArgumentNullException.ThrowIfNull(api);
        _api = api;
    }

    public async IAsyncEnumerable<string> StreamAllItemsAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        int pageNumber = 1;

        while (true)
        {
            ExternalApiPage<string> page = await _api
                .FetchPageAsync(pageNumber, cancellationToken)
                .ConfigureAwait(false);

            foreach (string item in page.Items)
            {
                yield return item;
            }

            if (!page.HasNextPage)
            {
                yield break;
            }

            pageNumber++;
        }
    }
}
```

### 5.5 Stop at a Processing Limit — `PartnerSyncService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using MyBackendApp.Infrastructure.ExternalApi;

namespace MyBackendApp.Api.Services;

public sealed class PartnerSyncService
{
    private readonly PartnerApiClient _client;

    public PartnerSyncService(PartnerApiClient client)
    {
        ArgumentNullException.ThrowIfNull(client);
        _client = client;
    }

    public async Task SyncItemsUntilLimitAsync(
        int maxItemsToProcess,
        CancellationToken cancellationToken = default)
    {
        if (maxItemsToProcess <= 0)
        {
            return;
        }

        int processedCount = 0;
        await using IAsyncEnumerator<string> enumerator = _client
            .StreamAllItemsAsync()
            .GetAsyncEnumerator(cancellationToken);

        while (processedCount < maxItemsToProcess && await enumerator.MoveNextAsync())
        {
            Console.WriteLine($"Processing item: {enumerator.Current}");
            processedCount++;
        }
    }
}
```

The loop checks the limit **before** asking the iterator for another item, so it doesn't fetch an extra page after reaching the limit. If the limit is reached partway through a page, that current page has already been fetched and buffered; the next page isn't requested. The `await using` disposes the async enumerator on completion or early exit. A normal `await foreach` is the idiomatic choice when consuming the whole sequence; explicit enumerator control is useful here to show the exact early-stop boundary.

## 6. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Iterator** | A method or accessor that defines how to produce a sequence incrementally. | Traverses computed or stored data without first requiring a complete result collection. |
| **`yield return`** | Provides one value and suspends iterator execution. | Lets downstream code pull values on demand. |
| **`yield break`** | Ends iteration before the iterator body reaches its end. | Implements conditional termination, such as stopping at a marker. |
| **Deferred execution** | Iterator-body work begins when enumeration advances, not when the sequence is obtained. | Avoids work when a sequence isn't consumed, but can repeat work on a second enumeration. |
| **State machine** | Compiler-generated machinery that preserves position and locals between yields. | Makes custom traversal possible without handwritten enumerator state. |
| **`IEnumerable<T>`** | Synchronous generic enumerable sequence contract. | Consumed with `foreach`; iterator execution is synchronous. |
| **`IAsyncEnumerable<T>`** | Asynchronous generic sequence contract. | Consumed with `await foreach`; suitable for asynchronous I/O pipelines. |
| **Streaming pipeline** | A producer and consumer process values incrementally. | Can bound memory, provided the source and intermediate stages don't buffer the full result. |

## 7. Official References

- [Iterators — C#](https://learn.microsoft.com/en-us/dotnet/csharp/iterators) — iterator methods, `foreach`, and synchronous/asynchronous sequence generation.
- [`yield` statement — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/yield) — `yield return`, `yield break`, deferred execution, resource cleanup, and language restrictions.
- [`IEnumerable<T>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1?view=net-10.0) — synchronous enumeration contract.
- [`IAsyncEnumerable<T>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1?view=net-10.0) — asynchronous enumeration contract.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is LTS.
