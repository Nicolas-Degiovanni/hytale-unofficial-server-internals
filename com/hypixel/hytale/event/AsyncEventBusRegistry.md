---
description: Architectural reference for AsyncEventBusRegistry
---

# AsyncEventBusRegistry

**Package:** com.hypixel.hytale.event
**Type:** Service / Registry

## Definition
```java
// Signature
public class AsyncEventBusRegistry<KeyType, EventType extends IAsyncEvent<KeyType>>
   extends EventBusRegistry<KeyType, EventType, AsyncEventBusRegistry.AsyncEventConsumerMap<EventType>> {
```

## Architecture & Concepts

The AsyncEventBusRegistry is a foundational component of the Hytale event system, designed for high-throughput, non-blocking event processing. It serves as the central dispatcher for events that may involve I/O operations, long-running computations, or other tasks that must not block the main engine thread.

Its architecture is a departure from traditional synchronous event buses. Instead of invoking listeners sequentially in a blocking loop, it constructs a pipeline of asynchronous operations using Java's **CompletableFuture**. Each registered listener is a function that receives a future and returns a new future, effectively chaining itself onto the event processing flow.

This model is critical for performance and scalability, especially in a networked game environment. It allows systems like asset loading, network packet handling, and database queries to subscribe to events without introducing latency or stalls into the core game loop.

The registry categorizes listeners into three distinct types, processed in a specific order:
1.  **Global Listeners:** These are invoked for every event dispatched through the registry, regardless of its key. They are suitable for cross-cutting concerns like logging, metrics, or global state validation.
2.  **Keyed Listeners:** These are the most common type, invoked only when an event with a specific, matching key is dispatched. This allows for targeted logic, such as handling an event for a particular player, entity, or world chunk.
3.  **Unhandled Listeners:** These are invoked as a fallback only if an event is dispatched for a key that has *no* registered keyed listeners. This is useful for implementing default behaviors or logging unhandled event scenarios.

## Lifecycle & Ownership

-   **Creation:** An AsyncEventBusRegistry is not instantiated directly. Instances are created and managed by a central service container or engine bootstrap process. Typically, one registry instance is created per unique event type (e.g., one for PlayerJoinEvent, another for ChunkLoadEvent).
-   **Scope:** The registry is a long-lived, session-scoped service. It persists for the entire lifetime of the client or server process.
-   **Destruction:** The registry can be shut down via an internal `shutdown` flag. This prevents new listeners from being registered, allowing for a graceful teardown. The object is garbage collected when the managing service container is destroyed during application shutdown.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable, consisting of concurrent maps that store event listeners. The parent class `EventBusRegistry` utilizes a `ConcurrentHashMap` to store listeners by their key and `CopyOnWriteArrayList` for the lists of listeners at each priority level. This structure is optimized for frequent dispatching (reads) and less frequent registrations (writes).
-   **Thread Safety:** The class is designed to be thread-safe for both registration and dispatching.
    -   Registering or unregistering listeners can occur from any thread without external synchronization. The use of copy-on-write collections ensures that dispatch operations in progress are not affected by concurrent modifications.
    -   Dispatching an event is also thread-safe and non-blocking. It immediately returns a `CompletableFuture` that represents the eventual result of the entire listener chain.

## API Surface

The public contract is focused on listener registration and obtaining a dispatcher.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerAsync(priority, key, function) | EventRegistration | O(N) | Registers an asynchronous, keyed listener. The function transforms a future, enabling chained async operations. N is the number of listeners at the given priority. |
| registerAsyncGlobal(priority, function) | EventRegistration | O(N) | Registers an asynchronous listener that fires for all events of this type. |
| registerAsyncUnhandled(priority, function) | EventRegistration | O(N) | Registers an asynchronous listener that fires only if no keyed listeners exist for a dispatched event. |
| dispatchFor(key) | IEventDispatcher | O(1) | Retrieves a dispatcher instance for a specific key. This is the primary entry point for firing an event. Returns a pre-optimized NO_OP dispatcher if no listeners exist. |
| register(priority, key, consumer) | EventRegistration | O(N) | A compatibility method to register a synchronous listener. The bus internally wraps the `Consumer` in an asynchronous `thenApply` stage. |

## Integration Patterns

### Standard Usage

The standard pattern involves retrieving the registry for a specific event type from a service context, registering one or more asynchronous listeners, and using the dispatcher to fire events.

```java
// 1. Obtain the registry for a specific event type (e.g., PlayerLoginEvent)
AsyncEventBusRegistry<UUID, PlayerLoginEvent> loginBus = serviceRegistry.get(PLAYER_LOGIN_BUS);

// 2. Register an async listener that performs a database lookup
loginBus.registerAsync(Priorities.NORMAL, eventFuture -> {
    return eventFuture.thenComposeAsync(event -> {
        // Non-blocking call to a data access layer
        return playerDataService.loadProfile(event.getPlayerId())
            .thenApply(profile -> {
                event.setPlayerProfile(profile);
                return event; // Pass the modified event down the chain
            });
    });
});

// 3. Elsewhere, dispatch an event
PlayerLoginEvent loginEvent = new PlayerLoginEvent(playerId);
CompletableFuture<PlayerLoginEvent> finalState = loginBus.dispatchFor(playerId).dispatch(loginEvent);

// 4. (Optional) Handle the final result of the entire chain
finalState.whenComplete((event, ex) -> {
    if (ex != null) {
        log.error("Failed to process login for " + event.getPlayerId(), ex);
    }
});
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new AsyncEventBusRegistry()`. This will create an isolated bus that the rest of the application is unaware of. Always retrieve the shared instance from the central service registry.
-   **Blocking in Listeners:** Never call blocking operations like `CompletableFuture.join()`, `get()`, or `Thread.sleep()` inside a listener function. This completely negates the benefit of the asynchronous architecture and can lead to thread starvation or deadlocks. Use `thenComposeAsync` or `thenApplyAsync` to chain non-blocking operations.
-   **Ignoring the Returned Future:** The `dispatch` method returns a `CompletableFuture` that resolves when all listeners in the chain have completed. Ignoring this future means you have no way to know when the event processing is finished or if it failed.

## Data Pipeline

The flow of an event through the registry is a deterministic, multi-stage asynchronous pipeline.

> Flow:
> Event Source -> `dispatchFor(key).dispatch(event)` -> Creates initial `CompletableFuture.completedFuture(event)` -> **AsyncEventBusRegistry** applies chained listeners -> `[Global Listeners]` -> `[Keyed Listeners]` -> `[Unhandled Listeners (if no keyed listeners were found)]` -> Returns final `CompletableFuture<EventType>` to caller

