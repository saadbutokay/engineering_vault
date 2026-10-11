Examples target **.NET 10 (LTS)** with nullable reference analysis enabled and **C# 14**.

## 1. Conceptual Foundation

### 1.1 What Is an Event?

An **event** is a delegate-based member used for publisher/subscriber notifications. The **publisher** raises the event; **subscribers** attach handlers and respond. The publisher doesn't need to know which objects are listening.

The `event` keyword adds an important access boundary:

| Public delegate field | Public event |
|---|---|
| External code can invoke it, replace its invocation list, or clear it. | External code can subscribe with `+=` and unsubscribe with `-=`. |
| The publisher loses control of who can raise the notification. | Only code in the declaring type (and permitted derived types) can raise it. |

Events are built on delegates, but they aren't durable messages or a distributed event bus. A conventional `EventHandler`-based event is an **in-process, synchronous** notification: handlers run as part of the publisher's call. For domain workflows or cross-service communication, distinguish C# events from explicit domain-event dispatch and durable integration messaging.

### 1.2 Why Events Matter in Backend Engineering

Events can support:

- In-process notifications when an object changes state.
- Decoupled handlers for logging, auditing, or cache invalidation.
- BCL and framework extension points, such as `FileSystemWatcher.Changed`.
- Domain-event designs, when event data and dispatch timing are managed deliberately.

For critical work such as sending email or publishing to another service, an event handler attached directly to an aggregate isn't automatically transactional, durable, asynchronous, or isolated from failures. A domain event dispatcher or an outbox/message-broker design may be more appropriate.

## 2. The Standard .NET Event Pattern

### 2.1 Declaring and Raising an Event

The usual .NET pattern uses `EventHandler` for an event without data and `EventHandler<TEventArgs>` for one with data. Its signature is `(object? sender, TEventArgs e)`. Event-argument types commonly derive from `EventArgs` by convention; the generic delegate in .NET 10 doesn't require that inheritance constraint.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Events;

public sealed class OrderShippedEventArgs : EventArgs
{
    public Guid OrderId { get; }
    public DateTimeOffset ShippedAtUtc { get; }

    public OrderShippedEventArgs(Guid orderId, DateTimeOffset shippedAtUtc)
    {
        OrderId = orderId;
        ShippedAtUtc = shippedAtUtc;
    }
}

public class OrderProcessor
{
    public event EventHandler<OrderShippedEventArgs>? OrderShipped;

    public void Ship(Guid orderId)
    {
        var args = new OrderShippedEventArgs(orderId, DateTimeOffset.UtcNow);
        OnOrderShipped(args);
    }

    // The protected virtual raiser follows the conventional extensibility pattern.
    protected virtual void OnOrderShipped(OrderShippedEventArgs e)
    {
        OrderShipped?.Invoke(this, e);
    }
}
```

### 2.2 Subscribing and Unsubscribing

Keep the delegate instance—or use a named method—when it may need to be removed later. Repeating an anonymous lambda expression at `-=` doesn't reliably identify the earlier subscription.

```csharp
using System;
using MyBackendApp.Events;

var processor = new OrderProcessor();

EventHandler<OrderShippedEventArgs> handler = (sender, e) =>
{
    Console.WriteLine($"Order {e.OrderId} shipped at {e.ShippedAtUtc}.");
};

processor.OrderShipped += handler;
processor.Ship(Guid.NewGuid());
processor.OrderShipped -= handler;
```

### 2.3 Null-Conditional Invocation

A field-like event has no handlers until a subscriber attaches one. The idiomatic raising form is:

```csharp
OrderShipped?.Invoke(this, args);
```

If there are no subscribers, the call is skipped. The null-conditional invocation also evaluates the delegate once, avoiding a race between a separate null check and the call. If another thread unsubscribes after that delegate snapshot is obtained, its handler can still run for the current invocation; unsubscription isn't a cancellation barrier. Subscriber exceptions still propagate, and a throwing handler prevents later handlers in the invocation list from running.

### 2.4 Custom `add` / `remove` Accessors

Field-like events are sufficient for most code. A custom event has no compiler-generated backing field, so the type must provide its own storage and accessors. Use custom accessors for special storage, forwarding, or synchronization; if you write them, you own their thread-safety behavior.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Events;

public sealed class LockedOrderEventSource
{
    private readonly object _eventGate = new();
    private EventHandler<OrderShippedEventArgs>? _orderShipped;

    public event EventHandler<OrderShippedEventArgs>? OrderShipped
    {
        add
        {
            lock (_eventGate)
            {
                _orderShipped += value;
            }
        }
        remove
        {
            lock (_eventGate)
            {
                _orderShipped -= value;
            }
        }
    }

    public void Publish(OrderShippedEventArgs args)
    {
        EventHandler<OrderShippedEventArgs>? handlers;
        lock (_eventGate)
        {
            handlers = _orderShipped;
        }

        // Invoke outside the lock to avoid running subscriber code while holding it.
        handlers?.Invoke(this, args);
    }
}
```

## 3. Event Subscription Lifetimes and Memory

A delegate for an instance method holds a strong reference to its target. If a long-lived publisher (for example, a singleton service or static event) remains reachable, its event subscription can keep a short-lived subscriber alive. Unsubscribe when the subscription is no longer needed—often from an `IDisposable.Dispose()` method—or store the handler so it can be removed later.

A publisher/subscriber cycle by itself isn't necessarily a leak if the entire cycle is unreachable; the common problem is a reachable, long-lived publisher rooting a subscriber that should otherwise be collectible. Lambdas can also retain captured objects. Event handlers should generally be quick and non-blocking; slow work can delay the publisher, and an exception can prevent subsequent handlers from running.

## 4. Basic Syntax Example

This inventory example raises an event only after a successful quantity change. The item itself doesn't know that one subscriber logs changes while another checks for low stock.

```csharp
#nullable enable
using System;

namespace MyBackendApp.Basics;

public sealed class StockLevelChangedEventArgs : EventArgs
{
    public string Sku { get; }
    public int NewQuantity { get; }

    public StockLevelChangedEventArgs(string sku, int newQuantity)
    {
        Sku = sku;
        NewQuantity = newQuantity;
    }
}

public sealed class InventoryItem
{
    private int _quantity;

    public string Sku { get; }
    public event EventHandler<StockLevelChangedEventArgs>? StockLevelChanged;

    public InventoryItem(string sku, int initialQuantity)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(sku);
        if (initialQuantity < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(initialQuantity));
        }

        Sku = sku;
        _quantity = initialQuantity;
    }

    public void AdjustStock(int delta)
    {
        int newQuantity = checked(_quantity + delta);
        if (newQuantity < 0 || newQuantity == _quantity)
        {
            if (newQuantity < 0)
            {
                throw new InvalidOperationException("Stock quantity can't be negative.");
            }

            return;
        }

        _quantity = newQuantity;
        StockLevelChanged?.Invoke(this, new StockLevelChangedEventArgs(Sku, _quantity));
    }
}

public static class EventDemo
{
    public static void Run()
    {
        var item = new InventoryItem("SKU-100", initialQuantity: 50);

        item.StockLevelChanged += (sender, e) =>
            Console.WriteLine($"[Log] Stock for {e.Sku} changed to {e.NewQuantity}");

        item.StockLevelChanged += (sender, e) =>
        {
            if (e.NewQuantity < 10)
            {
                Console.WriteLine($"[Alert] Low stock warning for {e.Sku}!");
            }
        };

        item.AdjustStock(-45); // Both handlers run; new quantity is 5.
    }
}
```

`InventoryItem` is a simple, single-threaded example. If multiple threads can change the same item, protect its state and define how event delivery is synchronized. Raising after updating state also means a subscriber exception doesn't roll the state back.

## 5. Backend Example: In-Process Order Status Notifications

The following files demonstrate ordinary C# events for in-process observers. They are **not** a durable domain-event dispatcher or integration-event implementation. A production aggregate often records explicit domain-event objects and dispatches them at a deliberate persistence/transaction boundary. To notify other services reliably after commit, use an asynchronous integration event and an outbox or comparable messaging design.

The declarations are split into files because a single C# source file can't contain several file-scoped namespace declarations (`namespace Name;`).

### 5.1 `Order.cs`

```csharp
#nullable enable
using System;

namespace MyBackendApp.Core.Domain.Orders;

public enum OrderStatus
{
    Draft,
    Paid,
    Shipped,
    Cancelled
}

public sealed class OrderStatusChangedEventArgs : EventArgs
{
    public Guid OrderId { get; }
    public OrderStatus PreviousStatus { get; }
    public OrderStatus NewStatus { get; }
    public DateTimeOffset ChangedAtUtc { get; }

    public OrderStatusChangedEventArgs(
        Guid orderId,
        OrderStatus previousStatus,
        OrderStatus newStatus,
        DateTimeOffset changedAtUtc)
    {
        OrderId = orderId;
        PreviousStatus = previousStatus;
        NewStatus = newStatus;
        ChangedAtUtc = changedAtUtc;
    }
}

public sealed class Order
{
    public Guid Id { get; }
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;

    public event EventHandler<OrderStatusChangedEventArgs>? StatusChanged;

    public Order(Guid id)
    {
        Id = id;
    }

    public void TransitionTo(OrderStatus newStatus)
    {
        if (Status == newStatus)
        {
            return;
        }

        OrderStatus previousStatus = Status;
        Status = newStatus;

        var args = new OrderStatusChangedEventArgs(
            Id,
            previousStatus,
            newStatus,
            DateTimeOffset.UtcNow);

        StatusChanged?.Invoke(this, args);
    }
}
```

This small aggregate assumes transitions are serialized. A concurrently accessed aggregate needs synchronization around state changes, and subscriber code should not run while holding a state lock.

### 5.2 `CustomerNotificationService.cs`

```csharp
using System;
using MyBackendApp.Core.Domain.Orders;

namespace MyBackendApp.Infrastructure.Notifications;

public sealed class CustomerNotificationService
{
    public void SubscribeTo(Order order)
    {
        ArgumentNullException.ThrowIfNull(order);
        order.StatusChanged += HandleOrderStatusChanged;
    }

    public void UnsubscribeFrom(Order order)
    {
        ArgumentNullException.ThrowIfNull(order);
        order.StatusChanged -= HandleOrderStatusChanged;
    }

    private void HandleOrderStatusChanged(object? sender, OrderStatusChangedEventArgs e)
    {
        Console.WriteLine(
            $"[Notification] Order {e.OrderId}: {e.PreviousStatus} -> {e.NewStatus}.");
    }
}
```

### 5.3 `OrderAuditLogger.cs`

The audit logger uses a named handler so it can unsubscribe; an anonymous lambda that wasn't saved would be difficult to remove. The lock below protects the example's in-memory list if different orders raise events concurrently.

```csharp
#nullable enable
using System;
using System.Collections.Generic;
using System.Linq;
using MyBackendApp.Core.Domain.Orders;

namespace MyBackendApp.Infrastructure.Auditing;

public sealed class OrderAuditLogger
{
    private readonly object _gate = new();
    private readonly List<string> _auditTrail = [];

    public void SubscribeTo(Order order)
    {
        ArgumentNullException.ThrowIfNull(order);
        order.StatusChanged += HandleOrderStatusChanged;
    }

    public void UnsubscribeFrom(Order order)
    {
        ArgumentNullException.ThrowIfNull(order);
        order.StatusChanged -= HandleOrderStatusChanged;
    }

    public IReadOnlyList<string> GetAuditTrail()
    {
        lock (_gate)
        {
            return Array.AsReadOnly(_auditTrail.ToArray());
        }
    }

    private void HandleOrderStatusChanged(object? sender, OrderStatusChangedEventArgs e)
    {
        string entry = $"{e.ChangedAtUtc:O} | Order {e.OrderId} | {e.PreviousStatus} -> {e.NewStatus}";
        lock (_gate)
        {
            _auditTrail.Add(entry);
        }
    }
}
```

### 5.4 `OrderEventDemo.cs`

```csharp
using System;
using MyBackendApp.Core.Domain.Orders;
using MyBackendApp.Infrastructure.Auditing;
using MyBackendApp.Infrastructure.Notifications;

namespace MyBackendApp.Api;

public static class OrderEventDemo
{
    public static void Run()
    {
        var order = new Order(Guid.NewGuid());
        var notifier = new CustomerNotificationService();
        var auditor = new OrderAuditLogger();

        notifier.SubscribeTo(order);
        auditor.SubscribeTo(order);

        order.TransitionTo(OrderStatus.Paid);
        order.TransitionTo(OrderStatus.Shipped);

        foreach (string entry in auditor.GetAuditTrail())
        {
            Console.WriteLine(entry);
        }

        notifier.UnsubscribeFrom(order);
        auditor.UnsubscribeFrom(order);
    }
}
```

These subscribers are independent of the `Order` implementation, but they still run synchronously on the thread that calls `TransitionTo`. For real email, database, or broker I/O, don't assume a C# event provides background execution, retries, ordering across processes, or durable delivery.

## 6. Key Terms Summary

| Term | Definition | Backend relevance |
|---|---|---|
| **Event** | A delegate-based member that lets the declaring type publish notifications. | Supports in-process observer relationships with controlled subscription. |
| **Publisher** | The type that declares and raises the event. | Signals a state change without directly calling each subscriber. |
| **Subscriber** | A method attached to an event, usually with `+=`. | Reacts to an event and should unsubscribe when its lifetime ends. |
| **`EventHandler<TEventArgs>`** | A standard delegate with `(object? sender, TEventArgs e)` parameters. | Gives event declarations a consistent .NET handler signature. |
| **`?.Invoke`** | Null-conditional delegate invocation. | Skips the call if there are no handlers and evaluates the delegate once. |
| **Custom event accessors** | User-defined `add` and `remove` storage/forwarding logic. | Useful for special event storage; the type must implement its own synchronization. |
| **Subscription leak** | A long-lived publisher retains a subscriber through the handler delegate. | Motivates removing subscriptions, often during `Dispose()`. |
| **Domain event** | An explicit notification that a meaningful domain action occurred. | Can decouple same-domain side effects when dispatch and transaction timing are deliberate. |
| **Integration event** | An asynchronous message published after a state change is committed. | Communicates reliably across process or service boundaries, typically with an outbox. |

## 7. Official References

- [The `event` keyword — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/event) — event access, publisher invocation, and C# event features.
- [Handling and raising events — .NET](https://learn.microsoft.com/en-us/dotnet/standard/events/) — the .NET observer/delegate pattern and conventional raiser methods.
- [`EventHandler<TEventArgs>` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.eventhandler-1?view=net-10.0) — standard generic event handler delegate.
- [Member access and null-conditional operators — C# reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators) — null-conditional invocation and delegate safety.
- [Domain events: design and implementation — .NET](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation) — domain events versus integration events and dispatch considerations.
- [`FileSystemWatcher.Changed` API for .NET 10](https://learn.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher.changed?view=net-10.0) — a BCL event example.
- [C# language versioning](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning) — .NET 10 defaults to C# 14.
- [.NET releases, patches, and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support) — current support tracks; .NET 10 is LTS.
