Examples target **.NET 10 (LTS)** with nullable reference analysis enabled. The .NET 10 SDK defaults to **C# 14** unless the project overrides `LangVersion`.

## 1. Conceptual Foundation

### 1.1 What Are Generics?

**Generics** let classes, structs, interfaces, delegates, and methods declare **type parameters**—placeholders for types supplied when the generic API is used. For example, `T` in `List<T>` is a type parameter; `int` in `List<int>` is a type argument. The compiler checks that values match the chosen type, so callers don't need to cast every item back from `object`.

Before generic APIs, reusable collections often stored `object`. Such a collection could accept incompatible values and require runtime casts; storing a value type through an object-based API also commonly boxes it. A typed generic collection avoids those problems in ordinary typed use:

| Non-generic collection | Generic collection |
|---|---|
| `ArrayList list = new();` | `List<int> list = new();` |
| `list.Add(42);` boxes the `int` when stored as `object`. | `list.Add(42);` stores a typed `int` without that boxing. |
| `list.Add("hello");` compiles, even if callers expect integers. | `list.Add("hello");` doesn't compile for `List<int>`. |
| Retrieving an item requires a cast. | Retrieving an item has the declared type. |

Generics **reduce** casts and avoid boxing on common value-type paths; they don't eliminate boxing or allocations everywhere. Boxing can still occur if code converts a value to `object` or uses a non-generic API, and generic collection operations can still allocate for reasons such as capacity growth.

### 1.2 Why Generics Matter in Backend Engineering

Generics support reusable, type-safe building blocks such as:

- Collections: `List<T>`, `Dictionary<TKey,TValue>`, and `IEnumerable<T>`.
- Response wrappers: `Result<T>`, `ApiResponse<T>`, and `PagedResult<T>`.
- Reusable algorithms: filtering, mapping, and selection methods.
- Contracts and infrastructure: for example, `IRepository<TEntity>` or serializers parameterized by a model type.
- Constraints that let reusable code require a base class, interface, or constructor capability.

A generic repository is **one possible design**, not a requirement for every backend. Frameworks such as EF Core already expose generic APIs; adding a repository abstraction should solve a concrete design or testing need rather than merely add another layer.

## 2. Generic Types and Generic Methods

### 2.1 Generic Classes

A generic class can use its type parameter in fields, method parameters, and return types. This example returns a detached read-only **shallow snapshot**: the list structure is copied, but reference-type items themselves aren't cloned.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

public sealed class Repository<T>
{
    private readonly List<T> _items = [];

    public void Add(T item) => _items.Add(item);

    public IReadOnlyList<T> GetAll() => Array.AsReadOnly(_items.ToArray());
}
```

### 2.2 Generic Methods

A generic method declares its own type parameter, independently of any type parameters on its containing class. The compiler can often infer the type argument from the method arguments.

```csharp
#nullable enable
using System;
using System.Collections.Generic;

public static class CollectionHelpers
{
    public static T? FirstOrDefaultMatch<T>(
        IEnumerable<T> source,
        Func<T, bool> predicate)
    {
        ArgumentNullException.ThrowIfNull(source);
        ArgumentNullException.ThrowIfNull(predicate);

        foreach (T item in source)
        {
            if (predicate(item))
            {
                return item;
            }
        }

        return default;
    }
}

// T is inferred as string from the first argument.
string? match = CollectionHelpers.FirstOrDefaultMatch(
    new[] { "Amina", "Rafi", "Maya" },
    name => name.StartsWith("M", StringComparison.Ordinal));
```

For an unconstrained `T`, `T?` annotates a possibly-null result when `T` is a reference type. If `T` is a value type, an unsuccessful search returns that type's `default` value (for example, `0` for `int`), which may not uniquely mean “not found.” Use a result type or a separate success flag when that distinction matters.

### 2.3 Multiple Type Parameters

A generic type can have more than one type parameter. `notnull` communicates that the key argument shouldn't be nullable in nullable-aware code; it isn't a runtime validation mechanism.

```csharp
using System.Collections.Generic;

public sealed class KeyValueStore<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _store = new();

    public void Set(TKey key, TValue value) => _store[key] = value;

    public TValue Get(TKey key) => _store[key];
}
```

## 3. Generic Constraints

Constraints describe what a type argument must support, so the compiler can permit operations beyond those available on `object`. Constraints apply to generic types and methods.

| Constraint | Meaning and important detail |
|---|---|
| `where T : struct` | Requires a non-nullable value type. It implies `new()` and can't be combined with `new()` or `unmanaged`. |
| `where T : class` | Requires a reference type; in a nullable-enabled context, it means a non-nullable reference type. |
| `where T : class?` | Allows nullable or non-nullable reference types. |
| `where T : notnull` | Requires a non-nullable type according to nullable analysis. Violations are reported as compiler warnings; this isn't a runtime null check. |
| `where T : unmanaged` | Requires a non-nullable unmanaged value type (one whose fields are also unmanaged). It implies `struct` and can't be combined with `struct` or `new()`. |
| `where T : BaseClass` | Requires `T` to be or derive from the specified base class. Nullable-aware code can use `BaseClass?` when a nullable reference argument is intended. |
| `where T : IInterface` | Requires `T` to implement the interface. Multiple interface constraints are allowed; nullable-aware code can use `IInterface?` when a nullable reference argument is intended. |
| `where T : U` | Requires the type argument for `T` to be or derive from the type supplied for type parameter `U`. |
| `where T : new()` | Requires a public parameterless constructor, enabling `new T()`. When combined with other constraints, it goes last (except for an applicable `allows ref struct` anti-constraint). |
| `where T : default` | Advanced: resolves a constraint ambiguity on an override or explicit interface implementation; it isn't a general type filter. |
| `where T : allows ref struct` | C# 13+ anti-constraint: permits a `ref struct` type argument. The generic implementation must obey ref-safety rules for that possibility. |

Constraints have ordering and combination rules. For example, `struct` and `unmanaged` imply constructor capability and can't be combined with `new()`; a base-class constraint can't be combined with `class`, `class?`, `struct`, `notnull`, or `unmanaged`. See the official constraints reference for the complete rules.

```csharp
using System;

public interface IEntity
{
    Guid Id { get; }
}

// The order is intentional: new() follows the other constraints.
public sealed class EntityFactory<T> where T : class, IEntity, new()
{
    public T CreateDefault() => new T();
}
```

## 4. Variance in Generic Interfaces and Delegates (`in` / `out`)

**Variance** permits certain implicit conversions between constructed interface or delegate types when their type arguments are related by inheritance:

- **Covariance (`out T`)** permits a constructed type using a more-derived type argument to be used where the corresponding base-type construction is expected. The type parameter must be used in output-safe positions.
- **Contravariance (`in T`)** permits a constructed type using a base type argument to be used where the corresponding derived-type construction is expected. The type parameter must be used in input-safe positions.

Variance is available only on generic interfaces and delegates, not generic classes such as `List<T>`. Variance conversions apply to reference types; they don't make `List<Derived>` convertible to `List<Base>`, and don't provide the same conversion for value-type arguments.

```csharp
using System;

public class Person { }
public sealed class Employee : Person { }

public interface IReadOnlyRepository<out T>
{
    T GetById(Guid id); // T appears in an output position.
}

public interface IValidator<in T>
{
    bool Validate(T item); // T appears in an input position.
}

public static class VarianceDemo
{
    public static void ShowAssignments(
        IReadOnlyRepository<Employee> employeeRepository,
        IValidator<Person> personValidator)
    {
        IReadOnlyRepository<Person> personRepository = employeeRepository;
        IValidator<Employee> employeeValidator = personValidator;
        _ = personRepository;
        _ = employeeValidator;
    }
}
```

The assignments are type-safe: an employee repository returns employees that are also people, and a validator that accepts any person can validate an employee. `IEnumerable<out T>`, `Func<in T, out TResult>`, and `Action<in T>` are familiar framework examples of variance.

## 5. Basic Syntax Example

This complete example uses a type parameter constrained to reference types implementing `IEntity`.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Basics;

public interface IEntity
{
    Guid Id { get; }
}

public sealed class InMemoryRepository<T> where T : class, IEntity
{
    private readonly Dictionary<Guid, T> _storage = new();

    public void Add(T entity)
    {
        ArgumentNullException.ThrowIfNull(entity);
        _storage[entity.Id] = entity;
    }

    public T? FindById(Guid id)
    {
        return _storage.TryGetValue(id, out T? entity) ? entity : null;
    }

    public IReadOnlyList<T> GetAll() => Array.AsReadOnly(_storage.Values.ToArray());
}

public sealed class Customer : IEntity
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
}

public static class GenericsDemo
{
    public static void Run()
    {
        var repository = new InMemoryRepository<Customer>();
        var customer = new Customer { Id = Guid.NewGuid(), Name = "Alice" };

        repository.Add(customer);

        Customer? found = repository.FindById(customer.Id);
        Console.WriteLine(found?.Name ?? "Not Found");
    }
}
```

`GetAll` copies and wraps the collection structure, but the `Customer` objects remain shared references. The repository is also an in-memory example, not a synchronized or durable store.

## 6. Backend Example: Generic Paging and Repository Contracts

Generics make reusable response wrappers and contracts possible. The example below is deliberately split into **separate files**: putting several file-scoped namespace declarations (`namespace Name;`) into one C# file, as in a single combined snippet, is invalid. These files can be placed in the same project.

A generic repository abstraction is an option, not automatically the best EF Core design. The in-memory implementation is for illustration/testing only: it is not EF Core, isn't thread-safe, and sorts/materializes all stored entities before paging. A database-backed implementation should apply filtering, ordering, counting, and pagination in the database and honor cancellation during I/O.

### 6.1 `PagedResult<T>.cs`

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyBackendApp.Core.Common;

public sealed class PagedResult<T>
{
    private PagedResult(T[] items, int pageNumber, int pageSize, int totalCount)
    {
        Items = Array.AsReadOnly(items);
        PageNumber = pageNumber;
        PageSize = pageSize;
        TotalCount = totalCount;
    }

    public IReadOnlyList<T> Items { get; }
    public int PageNumber { get; }
    public int PageSize { get; }
    public int TotalCount { get; }

    public int TotalPages => TotalCount == 0
        ? 0
        : (int)(((long)TotalCount + PageSize - 1) / PageSize);

    public static PagedResult<T> Create(
        IReadOnlyList<T> source,
        int pageNumber,
        int pageSize)
    {
        ArgumentNullException.ThrowIfNull(source);

        if (pageNumber <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(pageNumber));
        }

        if (pageSize <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(pageSize));
        }

        int totalCount = source.Count;
        long offset = ((long)pageNumber - 1) * pageSize;
        T[] pageItems = offset >= totalCount
            ? Array.Empty<T>()
            : source.Skip((int)offset).Take(pageSize).ToArray();

        return new PagedResult<T>(pageItems, pageNumber, pageSize, totalCount);
    }
}
```

`Create` uses the enclosing `PagedResult<T>` type parameter; it is **not itself a generic method** because it declares no separate `<T>` parameter list. `Items` is a copied, read-only collection surface, but items that are reference types aren't deep-cloned.

### 6.2 `IEntity.cs`

```csharp
using System;

namespace MyBackendApp.Core.Domain.Common;

public interface IEntity
{
    Guid Id { get; }
}
```

### 6.3 `IRepository<TEntity>.cs`

```csharp
#nullable enable
using System;
using System.Threading;
using System.Threading.Tasks;
using MyBackendApp.Core.Common;
using MyBackendApp.Core.Domain.Common;

namespace MyBackendApp.Core.Interfaces;

public interface IRepository<TEntity> where TEntity : class, IEntity
{
    Task<TEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<PagedResult<TEntity>> GetPageAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default);
    Task AddAsync(TEntity entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(TEntity entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}
```

### 6.4 `InMemoryRepository<TEntity>.cs` — illustrative implementation

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using MyBackendApp.Core.Common;
using MyBackendApp.Core.Domain.Common;
using MyBackendApp.Core.Interfaces;

namespace MyBackendApp.Infrastructure.Persistence;

public sealed class InMemoryRepository<TEntity> : IRepository<TEntity>
    where TEntity : class, IEntity
{
    private readonly Dictionary<Guid, TEntity> _store = new();

    public Task<TEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        _store.TryGetValue(id, out TEntity? entity);
        return Task.FromResult(entity);
    }

    public Task<PagedResult<TEntity>> GetPageAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();

        TEntity[] orderedSnapshot = _store.Values
            .OrderBy(entity => entity.Id)
            .ToArray();

        PagedResult<TEntity> page = PagedResult<TEntity>.Create(
            orderedSnapshot,
            pageNumber,
            pageSize);

        return Task.FromResult(page);
    }

    public Task AddAsync(TEntity entity, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        ArgumentNullException.ThrowIfNull(entity);
        _store.Add(entity.Id, entity);
        return Task.CompletedTask;
    }

    public Task UpdateAsync(TEntity entity, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        ArgumentNullException.ThrowIfNull(entity);

        if (!_store.ContainsKey(entity.Id))
        {
            throw new KeyNotFoundException($"Entity with Id '{entity.Id}' was not found.");
        }

        _store[entity.Id] = entity;
        return Task.CompletedTask;
    }

    public Task DeleteAsync(Guid id, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        _store.Remove(id);
        return Task.CompletedTask;
    }
}
```

The methods return completed `Task` objects because this in-memory sample performs no asynchronous I/O. A real database implementation would use its provider's asynchronous query/save APIs; adding `async` to a method that only wraps immediate in-memory work doesn't make that work asynchronous.

### 6.5 `Product.cs`

```csharp
using System;
using MyBackendApp.Core.Domain.Common;

namespace MyBackendApp.Core.Domain.Products;

public sealed class Product : IEntity
{
    public Guid Id { get; init; }
    public string Sku { get; init; } = string.Empty;
    public decimal Price { get; init; }
}
```

### 6.6 `ProductCatalogService.cs`

```csharp
#nullable enable
using System;
using System.Threading;
using System.Threading.Tasks;
using MyBackendApp.Core.Common;
using MyBackendApp.Core.Domain.Products;
using MyBackendApp.Core.Interfaces;

namespace MyBackendApp.Api.Services;

public sealed class ProductCatalogService
{
    private readonly IRepository<Product> _productRepository;

    public ProductCatalogService(IRepository<Product> productRepository)
    {
        _productRepository = productRepository
            ?? throw new ArgumentNullException(nameof(productRepository));
    }

    public Task<PagedResult<Product>> GetProductPageAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        return _productRepository.GetPageAsync(pageNumber, pageSize, cancellationToken);
    }
}
```

For database-backed paging, use a stable business ordering (often with a unique tie-breaker) and apply `Skip`/`Take` and the total-count query at the data source. Sorting every row and paging in memory, as this teaching implementation does, does not scale to large tables.

## 7. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Type parameter** | A placeholder such as `T` declared by a generic type or method. | Lets a reusable component operate on a chosen type. |
| **Type argument** | The concrete type supplied for a parameter, such as `Customer` in `Repository<Customer>`. | Determines the constructed generic type used by application code. |
| **Generic type definition** | The open declaration containing type parameters, such as `Dictionary<TKey,TValue>`. | Serves as a reusable template. |
| **Constructed generic type** | A generic type with type arguments supplied, such as `Dictionary<Guid, Product>`. | The concrete type used in a service or data layer. |
| **Generic method** | A method that declares its own type parameter list. | Supports reusable algorithms with compiler inference. |
| **Constraint (`where`)** | A capability or type restriction on a generic parameter. | Lets infrastructure safely depend on an interface, base class, or constructor. |
| **Covariance (`out`)** | An interface/delegate conversion from a more-derived type argument to a base-type construction. | Useful for read-only producers. |
| **Contravariance (`in`)** | An interface/delegate conversion from a base-type argument to a derived-type construction. | Useful for consumers such as validators and handlers. |
| **Generic repository** | A data access abstraction parameterized by entity type. | Can share data-access contracts, but should be weighed against framework features and query requirements. |
| **Boxing** | Wrapping a value type as an `object` reference. | Generic typed APIs often avoid this cost on value-type paths, but don't guarantee zero allocations everywhere. |

## 8. Official References

- [Generics in .NET](https://learn.microsoft.com/en-us/dotnet/standard/generics/) — generic definitions, type parameters, type safety, and reuse.
- [Constraints on type parameters](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters) — constraint forms, nullable variants, ordering, and anti-constraints.
- [Covariance and contravariance in generics](https://learn.microsoft.com/en-us/dotnet/standard/generics/covariance-and-contravariance) — variance conversions and restrictions.
- [Nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types) — nullable annotations and compiler analysis.
- [Collection expressions — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions) — C# 12+ collection-expression syntax used in the examples.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is an LTS release.
- [`List<T>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-10.0) — generic list API used in the examples.
- [`Dictionary<TKey,TValue>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2?view=net-10.0) — generic key/value storage API used in the examples.
