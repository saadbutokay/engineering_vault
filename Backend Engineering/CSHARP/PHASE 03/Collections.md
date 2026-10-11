Examples target **.NET 10** with nullable reference analysis enabled. With the SDK's normal defaults, .NET 10 uses C# 14 unless the project overrides `LangVersion`.

## 1. Conceptual Foundation

### 1.1 What Are Collections?

A **collection** groups related values so code can store, retrieve, enumerate, and update them. Common .NET collection namespaces include:

- **`System.Collections`** — legacy, non-generic collections such as `ArrayList` and `Hashtable`. They store elements as `object`, so value types are boxed and retrieval often requires casts. Prefer generic collections in new code.
- **`System.Collections.Generic`** — type-safe collections such as `List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, `Queue<T>`, and `Stack<T>`. This is the usual starting point for ordinary in-memory collections.
- **`System.Collections.Concurrent`** — types designed for safe concurrent access, such as `ConcurrentDictionary<TKey,TValue>` and `ConcurrentQueue<T>`. They don't automatically make a multi-step operation across several collections atomic.
- **`System.Collections.Immutable`** — collections whose existing instances aren't changed by update operations; those operations produce immutable results, often with structural sharing. A no-op may reuse an instance.
- **`System.Collections.ObjectModel`** — collection-related types and wrappers, including `ReadOnlyCollection<T>` and `ObservableCollection<T>`.

### 1.2 Selected Interface Relationships

The diagram shows common relationships, not every interface each class implements. In particular, `Queue<T>` and `Stack<T>` are enumerable but aren't index-based `IList<T>` collections.

```text
IEnumerable<T>
├── ICollection<T>
│   ├── IList<T>  ─────────────── List<T>
│   └── ISet<T>   ─────────────── HashSet<T>, SortedSet<T>
├── Queue<T>      (also IReadOnlyCollection<T>)
└── Stack<T>      (also IReadOnlyCollection<T>)

IDictionary<TKey,TValue>
  also enumerates ICollection<KeyValuePair<TKey,TValue>>
  ├── Dictionary<TKey,TValue>
  └── SortedDictionary<TKey,TValue>
```

- **`IEnumerable<T>`** is an iteration contract. It enables `foreach` and LINQ-to-Objects, but doesn't promise that data is already materialized in memory; a sequence may be generated lazily.
- **`ICollection<T>`** adds a count and collection operations, including mutation methods. An implementation can be read-only and reject mutation.
- **`IList<T>`** adds index-based access and ordering.
- **`ISet<T>`** models uniqueness according to an equality comparer.
- **`IDictionary<TKey,TValue>`** models key/value lookup and enumerates key/value pairs.

Generic collections avoid the boxing and cast costs of legacy `object`-based collections in ordinary typed use, and they detect type mismatches at compile time. Value types can still be boxed if code accesses them through an `object` or non-generic API.

## 2. Core Collection Types

### 2.1 `List<T>` — Resizable, Indexable Sequence

`List<T>` preserves insertion order and provides O(1) indexed access. Adding at the end is **amortized O(1)**; an individual add can take O(n) when capacity must grow and existing elements are copied. Inserting or removing at an arbitrary index is O(n) because later elements shift. Capacity-growth details are implementation details, not a fixed API guarantee.

```csharp
using System;
using System.Collections.Generic;

List<Guid> orderIds = new();
orderIds.Add(Guid.NewGuid());
orderIds.Insert(0, Guid.NewGuid());
orderIds.RemoveAt(1);
```

### 2.2 `Dictionary<TKey,TValue>` — Key/Value Lookup

A dictionary uses a key comparer (by default, `EqualityComparer<TKey>.Default`) with hash-based storage. Lookup, insertion, and removal are typically **O(1) on average**, but worst-case behavior can be O(n). Enumeration order isn't a sorted-order contract; use a sorted collection when you require comparer-based ordering.

Keys must have equality and hash-code behavior that stays consistent while they're stored. Use a custom `IEqualityComparer<TKey>` when the domain needs different comparison rules, and avoid mutating key state that participates in hashing or equality.

```csharp
using System;
using System.Collections.Generic;

var inventory = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
inventory["SKU-001"] = 50;

if (inventory.TryGetValue("sku-001", out int quantity))
{
    Console.WriteLine(quantity);
}
```

### 2.3 `HashSet<T>` — Unique Membership

`HashSet<T>` stores unique values according to its comparer. `Add` returns `false` when an equal value is already present. `Contains`, `Add`, and `Remove` are typically O(1) on average, with worst-case behavior affected by collisions and comparer cost. The type supports set operations such as `UnionWith`, `IntersectWith`, and `ExceptWith`; it doesn't promise sorted enumeration.

```csharp
using System;
using System.Collections.Generic;

Guid userId = Guid.NewGuid();
var activeUserIds = new HashSet<Guid>();
activeUserIds.Add(userId);
bool exists = activeUserIds.Contains(userId);
```

### 2.4 `Queue<T>` — First In, First Out

`Enqueue` adds at the back and `Dequeue` removes from the front. These operations are typically amortized O(1). Use `TryDequeue` when an empty queue is an ordinary case and you want to avoid an exception.

```csharp
using System.Collections.Generic;

var jobQueue = new Queue<string>();
jobQueue.Enqueue("ProcessPayment");
string nextJob = jobQueue.Dequeue();
```

### 2.5 `Stack<T>` — Last In, First Out

`Push` adds to the top and `Pop` removes from the top. These operations are typically amortized O(1). Use `TryPop` when an empty stack is expected.

```csharp
using System.Collections.Generic;

var undoStack = new Stack<string>();
undoStack.Push("EditField");
string lastAction = undoStack.Pop();
```

### 2.6 `SortedDictionary<TKey,TValue>` and `SortedSet<T>`

These collections enumerate keys or values in comparer order and provide O(log n) lookup/update operations. They are useful when ordered traversal is part of the requirement; they trade the average hash-table performance of `Dictionary` and `HashSet` for sorted operations. The comparison is determined by the default or supplied comparer.

## 3. Key Interfaces Behind Collections

| Interface | Contract | Significance |
|---|---|---|
| `IEnumerable<T>` | Exposes a generic enumerator for `foreach`. | Minimal sequence contract used by LINQ-to-Objects; may represent a deferred sequence. |
| `ICollection<T>` | Adds count and collection operations such as `Add`, `Remove`, and `Contains`. | Useful when callers need collection-level operations; mutation can be unsupported by a particular implementation. |
| `IList<T>` | Adds indexed access and ordering. | Fits sequence APIs where callers need positional access. |
| `ISet<T>` | Adds set-specific uniqueness and set operations. | Models membership and mathematical set behavior. |
| `IDictionary<TKey,TValue>` | Key-based access and enumeration of `KeyValuePair<TKey,TValue>`. | Models associative data such as configuration or inventory maps. |
| `IReadOnlyList<T>`, `IReadOnlyCollection<T>`, `IReadOnlyDictionary<TKey,TValue>` | Read-only member surfaces. | Restrict mutation through that interface, but don't by themselves make an underlying object immutable or thread-safe. |

When exposing internal data, return a detached snapshot or an immutable/read-only wrapper if callers must not affect the original collection. Returning a mutable `List<T>` as `IReadOnlyList<T>` only hides mutating methods at compile time; another reference to the list can still change it.

## 4. Performance Characteristics Summary

Big-O figures below are typical collection-operation costs, not end-to-end request guarantees. Hash-based costs are averages; actual performance depends on hashing, comparers, data distribution, and resizing.

| Collection | Add / enqueue / push | Remove / dequeue / pop | Lookup | Ordering |
|---|---:|---:|---:|---|
| `List<T>` | O(1) amortized at end; O(n) at a specific resize | O(n) for arbitrary removal | O(1) by index; O(n) by value | Insertion order |
| `Dictionary<TKey,TValue>` | O(1) average; O(n) worst case | O(1) average; O(n) worst case | O(1) average by key | No sorted-order guarantee |
| `HashSet<T>` | O(1) average; O(n) worst case | O(1) average; O(n) worst case | O(1) average for `Contains` | No sorted-order guarantee |
| `Queue<T>` | O(1) amortized `Enqueue` | O(1) amortized `Dequeue` | No indexed lookup | FIFO |
| `Stack<T>` | O(1) amortized `Push` | O(1) amortized `Pop` | No indexed lookup | LIFO |
| `SortedDictionary<TKey,TValue>` | O(log n) | O(log n) | O(log n) by key | Sorted by key comparer |
| `SortedSet<T>` | O(log n) | O(log n) | O(log n) membership | Sorted by element comparer |

## 5. Basic Syntax Example

```csharp
#nullable enable
using System;
using System.Collections.Generic;

namespace MyBackendApp.Basics;

public static class CollectionsDemo
{
    public static void Run()
    {
        // List<T>: insertion-ordered and indexable
        List<string> departments = ["Engineering", "Sales", "Finance"];
        departments.Add("Marketing");
        Console.WriteLine(departments[1]); // Sales

        // Dictionary<TKey, TValue>: key-based lookup
        Dictionary<string, decimal> exchangeRates = new(StringComparer.OrdinalIgnoreCase)
        {
            ["USD"] = 1.00m,
            ["EUR"] = 0.92m,
            ["GBP"] = 0.79m
        };

        if (exchangeRates.TryGetValue("eur", out decimal rate))
        {
            Console.WriteLine($"EUR rate: {rate}");
        }

        // HashSet<T>: Add reports whether this value was new
        HashSet<string> processedOrderIds = [];
        bool wasAdded = processedOrderIds.Add("ORD-1001");
        bool wasDuplicate = !processedOrderIds.Add("ORD-1001");
        Console.WriteLine($"Added: {wasAdded}, Duplicate: {wasDuplicate}");

        // Queue<T>: FIFO job processing
        Queue<string> jobQueue = new();
        jobQueue.Enqueue("SendEmail");
        jobQueue.Enqueue("GenerateInvoice");
        Console.WriteLine(jobQueue.Dequeue()); // SendEmail

        // Stack<T>: LIFO history
        Stack<string> actionHistory = new();
        actionHistory.Push("UpdateProfile");
        actionHistory.Push("ChangePassword");
        Console.WriteLine(actionHistory.Pop()); // ChangePassword
    }
}
```

Collection expressions such as `[...]` require C# 12 or later; the .NET 10 default is C# 14. If a project overrides `LangVersion`, verify that its setting supports the syntax.

## 6. Backend Example: In-Process Sliding-Window Admission Gate

This small example combines a `Dictionary` (client windows), `Queue` (request timestamps), and `HashSet` (accepted idempotency keys). A single lock protects the *combined operation* across these collections. It returns a detached, read-only load snapshot rather than exposing the internal dictionary.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Linq;

namespace MyBackendApp.Infrastructure.Throttling;

public sealed class SlidingWindowAdmissionGate
{
    private sealed class ClientWindow
    {
        public Queue<DateTimeOffset> RequestTimes { get; } = new();
    }

    private readonly Dictionary<string, ClientWindow> _windows = new(StringComparer.Ordinal);
    private readonly HashSet<(string ClientId, string IdempotencyKey)> _acceptedKeys = new();
    private readonly object _gate = new();
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    public SlidingWindowAdmissionGate(int maxRequests, TimeSpan windowSize)
    {
        if (maxRequests <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(maxRequests));
        }

        if (windowSize <= TimeSpan.Zero)
        {
            throw new ArgumentOutOfRangeException(nameof(windowSize));
        }

        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

    public IReadOnlyDictionary<string, int> GetCurrentLoadSnapshot()
    {
        lock (_gate)
        {
            DateTimeOffset cutoff = DateTimeOffset.UtcNow - _windowSize;
            foreach (ClientWindow window in _windows.Values)
            {
                PruneExpired(window, cutoff);
            }

            var snapshot = _windows.ToDictionary(
                entry => entry.Key,
                entry => entry.Value.RequestTimes.Count,
                StringComparer.Ordinal);

            return new ReadOnlyDictionary<string, int>(snapshot);
        }
    }

    // Returns false for either an already accepted idempotency key or a full window.
    public bool TryAcceptRequest(string clientId, string idempotencyKey)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(clientId);
        ArgumentException.ThrowIfNullOrWhiteSpace(idempotencyKey);

        lock (_gate)
        {
            DateTimeOffset now = DateTimeOffset.UtcNow;
            DateTimeOffset cutoff = now - _windowSize;

            if (!_windows.TryGetValue(clientId, out ClientWindow? window) || window is null)
            {
                window = new ClientWindow();
            }

            PruneExpired(window, cutoff);

            var key = (ClientId: clientId, IdempotencyKey: idempotencyKey);
            if (_acceptedKeys.Contains(key))
            {
                return false;
            }

            if (window.RequestTimes.Count >= _maxRequests)
            {
                return false;
            }

            window.RequestTimes.Enqueue(now);
            _windows[clientId] = window;
            _acceptedKeys.Add(key);
            return true;
        }
    }

    private static void PruneExpired(ClientWindow window, DateTimeOffset cutoff)
    {
        while (window.RequestTimes.Count > 0 && window.RequestTimes.Peek() <= cutoff)
        {
            window.RequestTimes.Dequeue();
        }
    }
}
```

This is an **in-process teaching example**, not a production rate limiter or durable idempotency store. Its client windows and accepted-key set need eviction/retention policies to avoid unbounded growth; the idempotency key is scoped to the client, but a real API may also need to scope it to the operation and store/replay the original result. `DateTimeOffset.UtcNow` can be affected by clock adjustments, and each application instance has separate in-memory state. For ASP.NET Core, evaluate the built-in rate-limiting middleware and load-test its policies; use shared infrastructure when limits must span multiple instances.

## 7. Design Guidance and Common Mistakes

- **Using `ArrayList` or `Hashtable` in new code:** prefer generic collections for compile-time type safety and to avoid boxing/casts for value types.
- **Assuming average O(1) means guaranteed O(1):** hash-based collections can degrade with collisions, expensive comparers, or poor key hash behavior.
- **Mutating a key after inserting it:** changes to state used by equality or hashing can make dictionary/set entries difficult to find.
- **Relying on dictionary or hash-set enumeration order:** those types don't promise sorted order. Choose a sorted collection when order is part of the requirement.
- **Treating `IReadOnlyList<T>` as deep immutability:** it hides mutation methods from a particular reference; it doesn't freeze the underlying collection or prevent another alias from changing it.
- **Assuming ordinary generic collections are safe for concurrent writes:** `List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, `Queue<T>`, and `Stack<T>` need synchronization if multiple threads mutate shared instances.
- **Using one concurrent collection to protect a multi-collection invariant:** thread-safe operations on a `ConcurrentDictionary` don't make an update to a dictionary, queue, and set atomic as a group. Use a lock or redesign the operation.
- **Assuming an in-memory rate limiter is global:** each process has independent state. Multi-instance deployments need a distributed policy/store or an appropriate gateway/middleware design.
- **Keeping deduplication keys forever:** production idempotency needs an explicit retention, persistence, and replay policy; an unbounded `HashSet<T>` will grow indefinitely.

## 8. Key Terms Summary

| Term | Definition | Purpose in backend development |
|---|---|---|
| **`List<T>`** | Resizable, insertion-ordered, indexable generic sequence. | General-purpose ordered storage and result materialization. |
| **`Dictionary<TKey,TValue>`** | Key/value collection with comparer-based lookup. | Fast average-case lookup for maps, indexes, and client state. |
| **`HashSet<T>`** | Collection of unique elements according to an equality comparer. | Membership tests, set operations, and bounded deduplication. |
| **`Queue<T>`** | FIFO collection. | Work queues and time-ordered window tracking. |
| **`Stack<T>`** | LIFO collection. | Undo operations and depth-first algorithms. |
| **`SortedDictionary<TKey,TValue>`** | Dictionary enumerated in key-comparer order. | Ordered key traversal with logarithmic operations. |
| **`IEnumerable<T>`** | Generic iteration contract. | Enables `foreach` and LINQ; doesn't guarantee eager materialization. |
| **`IReadOnlyDictionary<TKey,TValue>`** | Read-only interface for key/value lookup. | Restricts mutation through the interface; a snapshot or immutable wrapper is needed for stronger isolation. |
| **Boxing** | Representing a value type as an `object` reference. | Generic collections usually avoid it for typed element storage. |
| **Amortized complexity** | Average cost per operation over a sequence of operations, including occasional expensive growth. | Explains why a dynamic array's end-add is typically efficient despite occasional O(n) resizing. |

## 9. Official References

- [Collections and data structures — .NET](https://learn.microsoft.com/en-us/dotnet/standard/collections/) — generic/non-generic collections, common interfaces, and collection behavior.
- [Thread-safe collections — .NET](https://learn.microsoft.com/en-us/dotnet/standard/collections/thread-safe/) — concurrent collections and synchronization guidance.
- [`List<T>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-10.0) — indexed generic list.
- [`Dictionary<TKey,TValue>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2?view=net-10.0) — key/value collection.
- [`HashSet<T>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.hashset-1?view=net-10.0) — unique-value set.
- [`SortedDictionary<TKey,TValue>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.sorteddictionary-2?view=net-10.0) — comparer-ordered key/value collection.
- [Collection expressions — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions) — target-typed `[...]` syntax and supported collection targets.
- [Rate limiting middleware in ASP.NET Core 10](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-10.0) — built-in rate-limiting policies and middleware.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
