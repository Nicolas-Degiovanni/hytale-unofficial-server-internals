---
description: Architectural reference for EventBus
---

# EventBus

**Package:** com.hypixel.hytale.event
**Type:** Transient

## Definition
```java
// Signature
public class EventBus implements IEventBus {
```

## Architecture & Concepts

The EventBus is the central messaging backbone for the Hytale engine, implementing a highly concurrent and flexible Publish-Subscribe pattern. Its primary architectural role is to decouple systems, allowing different engine components to communicate without holding direct references to one another.

The core design principle of the EventBus is that it does not manage listeners directly. Instead, it functions as a **manager of registries**. For each unique event type (e.g., PlayerJoinEvent), the EventBus lazily instantiates and maintains a dedicated `EventBusRegistry` instance. This segregation of responsibilities is critical for performance and concurrency.

A key architectural feature is the native distinction between synchronous and asynchronous events:
*   **Synchronous Events (IEvent):** These are dispatched immediately on the calling thread. They are handled by a `SyncEventBusRegistry`.
*   **Asynchronous Events (IAsyncEvent):** These are dispatched to a dedicated thread pool for processing, returning a `CompletableFuture`. They are managed by an `AsyncEventBusRegistry`.

This design allows the engine to safely perform long-running operations in response to events without blocking critical threads like the main game loop or network processing threads.

Furthermore, the system supports keyed and global listeners. Keyed listeners are only invoked for events matching a specific key (e.g., a specific entity ID), enabling highly targeted and efficient event handling.

### Lifecycle & Ownership
- **Creation:** The EventBus is not a self-managing singleton. It is designed to be instantiated once by a top-level application container, such as `ClientEntryPoint` or `ServerEntryPoint`, during the engine's bootstrap phase.
- **Scope:** A single EventBus instance persists for the entire application lifetime (e.g., a client session or a dedicated server process). It is a foundational service that must be available throughout the application lifecycle.
- **Destruction:** The `shutdown()` method **must** be invoked during the application's graceful shutdown sequence. This method iterates through all managed `EventBusRegistry` instances and calls their respective `shutdown()` methods, which is critical for terminating internal thread pools used by asynchronous event handlers.

**WARNING:** Failure to call `shutdown()` will result in resource leaks, specifically unterminated threads, which can prevent the JVM from exiting cleanly.

## Internal State & Concurrency
- **State:** The primary internal state is the `registryMap`, a map from an event's `Class` to its corresponding `EventBusRegistry`. This map is mutable and grows dynamically as new event types are introduced to the system at runtime.
- **Thread Safety:** This class is **thread-safe** and designed for high-concurrency environments.
    - The `registryMap` is a `ConcurrentHashMap`, which provides lock-free reads and high-performance, thread-safe writes.
    - The `computeIfAbsent` method is used to atomically create and insert new `EventBusRegistry` instances. This prevents race conditions where multiple threads might attempt to create a registry for the same event type simultaneously.
    - All public methods are safe to be called from any thread. The responsibility for managing listener concurrency is delegated to the underlying `SyncEventBusRegistry` and `AsyncEventBusRegistry` implementations.

## API Surface

The public API is designed for registering listeners and acquiring dispatchers to fire events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(...) | EventRegistration | O(1) amortized | Registers a synchronous event listener for a specific event type, key, and priority. |
| registerAsync(...) | EventRegistration | O(1) amortized | Registers an asynchronous event listener. The handler is a function that composes `CompletableFuture` chains. |
| registerGlobal(...) | EventRegistration | O(1) amortized | Registers a listener that is invoked for an event of a given type, regardless of its key. |
| registerUnhandled(...) | EventRegistration | O(1) amortized | Registers a listener that is invoked only if no other keyed or global listeners handle the event. |
| dispatchFor(class, key) | IEventDispatcher | O(1) | Retrieves a dispatcher for a specific synchronous event type and key. This is a lightweight factory method. |
| dispatchForAsync(class, key) | IEventDispatcher | O(1) | Retrieves a dispatcher for a specific asynchronous event type and key. |
| shutdown() | void | O(N) | Shuts down all managed registries. N is the number of unique event types registered. |

## Integration Patterns

### Standard Usage

The EventBus should always be retrieved from a central service locator or dependency injection container. Listeners are registered with a lambda or method reference.

```java
// Correctly obtain the shared EventBus instance
IEventBus eventBus = serviceContext.getService(IEventBus.class);

// Register a listener for a specific event type
EventRegistration<Void, PlayerConnectEvent> registration = eventBus.register(
    PlayerConnectEvent.class,
    event -> {
        // Logic to handle player connection
        LOGGER.info("Player {} has connected.", event.getPlayerName());
    }
);

// When the listener is no longer needed, it must be unregistered to prevent memory leaks
registration.unregister();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EventBus()` in application code. This creates an isolated bus that will not receive events from the rest of the engine, leading to silent failures and unpredictable behavior. Always use the shared, container-managed instance.
- **Blocking Synchronous Listeners:** Registering a long-running or blocking operation as a standard synchronous listener on a performance-critical event can cause severe application-wide performance degradation or freezes. Use `registerAsync` for any non-trivial work.
- **Ignoring EventRegistration:** The `register` methods return an `EventRegistration` object. Discarding this object makes it impossible to unregister the listener, which can lead to memory leaks if the listener holds references to objects with a shorter lifecycle.

## Data Pipeline

The EventBus acts as a central router, directing registration and dispatch requests to the appropriate specialized registry.

> **Registration Flow:**
> System Code -> `EventBus.register(PlayerEvent.class, ...)` -> `EventBus.getRegistry(PlayerEvent.class)` -> `(Atomic creation of SyncEventBusRegistry if absent)` -> `SyncEventBusRegistry.register(...)` -> Listener stored in priority list.

> **Dispatch Flow:**
> System Code -> `EventBus.dispatchFor(PlayerEvent.class, playerKey)` -> `(Returns a cached dispatcher)` -> `dispatcher.dispatch(event)` -> `SyncEventBusRegistry.dispatch(event)` -> Invokes all matching listeners in priority order.

