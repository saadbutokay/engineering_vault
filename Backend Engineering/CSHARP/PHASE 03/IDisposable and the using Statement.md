Examples target **.NET 10 (LTS)** and **C# 14**. Several disposal features shown here—`using` declarations and `IAsyncDisposable`—were introduced in C# 8 and remain supported.

## 1. Conceptual Foundation

### 1.1 The Problem: Resource Lifetimes

The garbage collector (GC) reclaims **managed memory** when objects are no longer reachable. It doesn't directly reclaim operating-system resources or unmanaged memory owned by those objects. Examples include native file and socket handles, unmanaged buffers, and some synchronization primitives.

Managed wrapper objects such as `FileStream`, `DbConnection`, and `Mutex` can implement `IDisposable` so their owners can release resources deterministically. Some wrappers also use `SafeHandle` or a finalizer as a last-resort safety net, but GC/finalizer timing is nondeterministic and isn't a substitute for prompt cleanup. Disposal can also release managed-but-limited resources or signal that a lock, transaction, or other scope has ended.

> **Ownership matters:** Dispose resources your code owns. Don't manually dispose an injected scoped service such as an EF Core `DbContext`; the dependency-injection container disposes it at the end of its registered lifetime.

`HttpClient` needs special lifetime management: don't create and dispose a new client for every request. Use a long-lived client with an appropriate pooled-connection lifetime or use `IHttpClientFactory`, which manages and pools message handlers. The right choice is about connection-pool lifetime, not just calling `Dispose` as often as possible.

### 1.2 The `IDisposable` Contract

`IDisposable` defines a single method, `Dispose()`. Calling it signals that an owned resource is no longer needed and gives the object a chance to release it immediately.

```csharp
// Simplified shape of the interface defined in System.
public interface IDisposable
{
    void Dispose();
}
```

If a class owns another `IDisposable` object, it normally cascades disposal to that owned object. If it merely borrows or receives an object it doesn't own, it must not dispose that object unless the ownership contract explicitly says otherwise.

## 2. The `using` Statement and Declaration

A `using` statement disposes its resource when control leaves its associated block. A **using declaration** has no explicit braces; it disposes at the end of the enclosing scope (often the method). Both forms dispose during ordinary exception unwinding as well as normal completion.

```csharp
using System.IO;

namespace MyBackendApp.Basics;

public static class UsingSyntaxDemo
{
    public static void WithUsingStatement(string path)
    {
        using (StreamReader reader = File.OpenText(path))
        {
            _ = reader.ReadLine();
        } // reader is disposed here.

        // Execution continues after the using block.
    }

    public static void WithUsingDeclaration(string path)
    {
        using StreamReader reader = File.OpenText(path);
        _ = reader.ReadLine();
    } // reader is disposed at the end of this method's scope.
}
```

Conceptually, a `using` statement is compiled into a `try`/`finally` that calls `Dispose()`:

```csharp
using System.IO;

namespace MyBackendApp.Basics;

public static class TryFinallyEquivalent
{
    public static void ReadFile(string path)
    {
        StreamReader reader = File.OpenText(path);
        try
        {
            _ = reader.ReadLine();
        }
        finally
        {
            reader?.Dispose();
        }
    }
}
```

The exact generated code depends on the resource type and nullability, but the important guarantee is cleanup when control leaves the scope. It doesn't cover abrupt process termination such as a forced kill or `Environment.FailFast`.

### 2.1 `await using`

Use `await using` for a resource that implements `IAsyncDisposable`. It awaits `DisposeAsync()` when the scope exits, including when an exception is propagating. `IAsyncDisposable.DisposeAsync()` returns a `ValueTask`.

## 3. Implementing Disposal Correctly

### 3.1 A Sealed Type That Owns a Managed Disposable

A sealed class that owns a managed disposable object normally needs a small, idempotent `Dispose` method and **no finalizer**. A finalizer is not needed merely because the owned `FileStream` ultimately uses an operating-system handle; the framework type handles that lower-level safety.

```csharp
#nullable enable
using System;
using System.IO;

namespace MyBackendApp.Resources;

public sealed class ManagedResourceHolder : IDisposable
{
    private FileStream? _stream;
    private bool _disposed;

    public ManagedResourceHolder(string path)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(path);
        _stream = new FileStream(path, FileMode.OpenOrCreate, FileAccess.Write);
    }

    public void WriteByte(byte value)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _stream!.WriteByte(value);
    }

    public void Dispose()
    {
        if (_disposed)
        {
            return;
        }

        _stream?.Dispose();
        _stream = null;
        _disposed = true;
    }
}
```

The guard makes repeated `Dispose()` calls harmless. Methods that require the resource can reject later use with `ObjectDisposedException`.

### 3.2 The Dispose Pattern, Finalizers, and `SafeHandle`

An inheritable class that implements `IDisposable` should generally provide a public, non-virtual `Dispose()` and a protected virtual `Dispose(bool disposing)` for derived classes. The synchronous method calls `Dispose(true)` and then `GC.SuppressFinalize(this)`; derived overrides should call the base implementation. If the type directly owns a raw unmanaged resource and can't use a safe-handle abstraction, a finalizer calls `Dispose(false)` as a last resort. The `disposing` flag distinguishes explicit disposal (where owned managed objects can also be disposed) from finalization (where managed-object cleanup must not be relied on).

**Prefer `SafeHandle` over writing a finalizer around a raw `IntPtr`.** `SafeHandle` handles finalization and native-handle lifetime rules; your owning class can dispose the safe handle deterministically and doesn't need its own finalizer just for that handle.

```csharp
using System;
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

namespace MyBackendApp.Resources;

public sealed class LocalBufferHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    private LocalBufferHandle()
        : base(ownsHandle: true)
    {
    }

    public static LocalBufferHandle Allocate(int byteCount)
    {
        if (byteCount <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(byteCount));
        }

        var handle = new LocalBufferHandle();
        handle.SetHandle(Marshal.AllocHGlobal(byteCount));
        return handle;
    }

    protected override bool ReleaseHandle()
    {
        Marshal.FreeHGlobal(handle);
        return true;
    }
}
```

`SafeHandle` implements `IDisposable` and has its own finalization safety net. Use `using` with the returned handle. Write a custom finalizer only for an advanced case where the type directly owns a raw unmanaged resource that cannot be represented by `SafeHandle`.

### 3.3 `IAsyncDisposable`

Implement `IAsyncDisposable` when cleanup itself needs asynchronous work, such as flushing an asynchronous stream. A type that can support both synchronous and asynchronous cleanup often implements both interfaces so callers can choose `using` or `await using` appropriately. For an inheritable async-disposable base class, follow the async dispose pattern with a protected virtual `DisposeAsyncCore()` that derived classes can override.

```csharp
using System;
using System.IO;
using System.Threading.Tasks;

namespace MyBackendApp.Resources;

public sealed class AsyncFileWriter : IDisposable, IAsyncDisposable
{
    private readonly StreamWriter _writer;

    public AsyncFileWriter(string path)
    {
        _writer = new StreamWriter(path);
    }

    public Task WriteLineAsync(string text) => _writer.WriteLineAsync(text);

    public void Dispose() => _writer.Dispose();

    public ValueTask DisposeAsync() => _writer.DisposeAsync();
}
```

```csharp
using System.Threading.Tasks;
using MyBackendApp.Resources;

namespace MyBackendApp.Basics;

public static class AsyncDisposeDemo
{
    public static async Task WriteAsync()
    {
        await using var writer = new AsyncFileWriter("async-output.txt");
        await writer.WriteLineAsync("Written before asynchronous disposal.");
    } // Awaits DisposeAsync() here.
}
```

If a class implements only `IAsyncDisposable`, consumers need to await `DisposeAsync()`—normally through `await using`—to dispose it.

## 4. Basic Syntax Example

```csharp
#nullable enable
using System;
using System.IO;

namespace MyBackendApp.Basics;

public sealed class TempFileWriter : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public TempFileWriter(string path)
    {
        _writer = new StreamWriter(path);
    }

    public void WriteLine(string content)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _writer.WriteLine(content);
    }

    public void Dispose()
    {
        if (_disposed)
        {
            return;
        }

        _writer.Dispose();
        _disposed = true;
        Console.WriteLine("TempFileWriter disposed and file handle released.");
    }
}

public static class DisposableDemo
{
    public static void Run()
    {
        // A using declaration disposes at the end of this method.
        using var writer = new TempFileWriter("output.txt");
        writer.WriteLine("Line 1");
        writer.WriteLine("Line 2");

        // A using statement disposes at the end of its explicit block.
        using (var blockScopedWriter = new TempFileWriter("output2.txt"))
        {
            blockScopedWriter.WriteLine("Scoped line");
        }

        Console.WriteLine("Continuing after the block-scoped resource was disposed.");
    }
}
```

The class owns its `StreamWriter`, so its `Dispose()` cascades to that writer. In a production class, consider whether the `Console.WriteLine` belongs in resource cleanup; disposal should generally release resources, not perform unrelated application logging.

## 5. Backend Example: Connections and Async Lock Handles

### 5.1 ADO.NET Repository — `OrderRepository.cs`

When application code creates a connection and command, it owns those objects and should dispose them. `await using` disposes them in reverse declaration order, including when `OpenAsync` or command execution fails.

```csharp
#nullable enable
using System;
using System.Data;
using System.Data.Common;
using System.Globalization;
using System.Threading;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.Persistence;

public sealed class OrderRepository
{
    private readonly Func<DbConnection> _connectionFactory;

    public OrderRepository(Func<DbConnection> connectionFactory)
    {
        ArgumentNullException.ThrowIfNull(connectionFactory);
        _connectionFactory = connectionFactory;
    }

    public async Task<decimal?> GetOrderTotalAsync(
        Guid orderId,
        CancellationToken cancellationToken)
    {
        await using DbConnection connection = _connectionFactory()
            ?? throw new InvalidOperationException("The connection factory returned null.");
        await connection.OpenAsync(cancellationToken);

        await using DbCommand command = connection.CreateCommand();
        command.CommandText = "SELECT TotalAmount FROM Orders WHERE Id = @OrderId";

        DbParameter parameter = command.CreateParameter();
        parameter.ParameterName = "@OrderId";
        parameter.DbType = DbType.Guid;
        parameter.Value = orderId;
        command.Parameters.Add(parameter);

        object? result = await command.ExecuteScalarAsync(cancellationToken);
        if (result is null || result is DBNull)
        {
            return null;
        }

        return Convert.ToDecimal(result, CultureInfo.InvariantCulture);
    }
}
```

Returning `null` distinguishes “no matching row” from a real total of zero. Parameter syntax and type mappings can vary by ADO.NET provider. With connection pooling enabled, disposing a connection usually closes the logical connection and returns it to the pool; the provider controls the physical connection lifetime.

### 5.2 Release a Lock Handle with `await using`

A disposable handle makes lock lifetime visible in code: when fulfillment returns or throws, the handle's `DisposeAsync()` is awaited. The provider below is an abstraction; a real distributed-lock implementation must use a cross-process service and provide suitable lease, expiry, and ownership semantics. An in-memory `HashSet<string>` is **not** a distributed lock.

```csharp
#nullable enable
using System;
using System.Threading;
using System.Threading.Tasks;

namespace MyBackendApp.Infrastructure.Concurrency;

public interface IDistributedLockProvider
{
    ValueTask<IAsyncDisposable?> TryAcquireAsync(
        string resourceKey,
        CancellationToken cancellationToken);
}

public sealed class OrderFulfillmentCoordinator
{
    private readonly IDistributedLockProvider _lockProvider;
    private readonly Func<Guid, CancellationToken, Task> _fulfillOrderAsync;

    public OrderFulfillmentCoordinator(
        IDistributedLockProvider lockProvider,
        Func<Guid, CancellationToken, Task> fulfillOrderAsync)
    {
        ArgumentNullException.ThrowIfNull(lockProvider);
        ArgumentNullException.ThrowIfNull(fulfillOrderAsync);
        _lockProvider = lockProvider;
        _fulfillOrderAsync = fulfillOrderAsync;
    }

    public async Task<bool> TryFulfillOrderAsync(
        Guid orderId,
        CancellationToken cancellationToken)
    {
        string lockKey = $"order-fulfillment:{orderId}";

        await using IAsyncDisposable? lockHandle =
            await _lockProvider.TryAcquireAsync(lockKey, cancellationToken);

        if (lockHandle is null)
        {
            return false;
        }

        await _fulfillOrderAsync(orderId, cancellationToken);
        return true;
    }
}
```

When `TryFulfillOrderAsync` exits its scope—whether it returns `true`, returns `false`, or propagates an exception—the acquired non-null handle is asynchronously disposed. The DI container disposes services it creates at the end of their configured lifetime; use `using` in application code for resources the current code explicitly owns.

## 6. Key Terms Summary

| Term | Definition | Purpose in backend development |
|---|---|---|
| **`IDisposable`** | Interface with `Dispose()` for deterministic cleanup. | Releases owned resources promptly rather than waiting for GC/finalizer timing. |
| **`using` statement** | A block-scoped construct that disposes a resource when control exits the block. | Keeps resource lifetime aligned with a specific operation. |
| **Using declaration** | A `using var` declaration disposed at the end of its enclosing scope. | Concisely manages method- or block-scoped resources. |
| **Dispose pattern** | The structure for an extensible disposable class, including `Dispose(bool)` where appropriate. | Coordinates cleanup for derived classes and owned resources. |
| **Finalizer** | A last-resort runtime callback for objects that directly own unmanaged resources. | A fallback only; not a replacement for deterministic disposal. |
| **`SafeHandle`** | A managed wrapper that owns a native handle and provides finalization safety. | Avoids writing fragile custom finalizers for common native handles. |
| **`IAsyncDisposable`** | Interface with asynchronous `DisposeAsync()` cleanup. | Supports cleanup that must await asynchronous work. |
| **Ownership** | Responsibility for creating and releasing a resource. | Prevents both leaks and disposing resources owned by DI or another component. |
| **Connection pool** | Provider-managed reuse of physical database or HTTP connections. | Disposal usually returns a logical connection to the pool; pooling behavior is provider-specific. |

## 7. Official References

- [`using` statement and declaration — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/using) — scope, disposal, `await using`, and try/finally behavior.
- [Implement a `Dispose` method — .NET](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) — idempotence, ownership, dispose patterns, finalizers, and `SafeHandle`.
- [Implement a `DisposeAsync` method — .NET](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-disposeasync) — asynchronous cleanup and async dispose patterns.
- [HttpClient guidelines for .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines) — connection-pool lifetimes, long-lived clients, and `IHttpClientFactory`.
- [Dependency injection guidelines — .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines#disposal-of-services) — how the container disposes services at the end of their lifetimes.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — support tracks; .NET 10 is LTS.
