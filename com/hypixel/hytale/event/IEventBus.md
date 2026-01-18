---
description: Architectural reference for IEventBus
---

# IEventBus

**Package:** com.hypixel.hytale.event
**Type:** Interface / Service Contract

## Definition
```java
// Signature
public interface IEventBus extends IEventRegistry {
```

## Architecture & Concepts
The IEventBus interface defines the central contract for the engine's publish-subscribe messaging system. It is the primary mechanism for decoupled, cross-system communication, allowing different engine components to interact without holding direct references to one another. This design is fundamental to the engine's modularity and extensibility.

At its core, the IEventBus facilitates a "fire-and-forget" communication model. A publisher dispatches an event object, and the bus is responsible for routing that event to any and all registered subscribers (listeners). This pattern prevents tight coupling; for example, the Physics system can dispatch a CollisionEvent without any knowledge of the systems that might react to it, such as the Audio system (to play a sound) or the Gameplay system (to apply damage).

The architecture explicitly supports both synchronous and asynchronous event dispatch. Synchronous events are handled immediately on the calling thread, while asynchronous events are processed on a background thread pool, returning a CompletableFuture to track completion. This dual-model is critical for performance, allowing low-latency, immediate operations to occur on the main thread while offloading heavier or non-critical tasks to prevent blocking the game loop.

The use of a generic KeyType in dispatch methods enables a more advanced, targeted form of eventing. Instead of broadcasting an event to all listeners of a certain type, a publisher can scope an event to a specific key (e.g., an entity ID). This allows listeners to subscribe to events for specific objects, creating a highly efficient and organized event flow.

## Lifecycle & Ownership
As an interface, IEventBus itself has no lifecycle. The documentation below pertains to its canonical implementation provided by the engine's service container.

- **Creation:** The concrete EventBus implementation is a foundational service, instantiated once by the primary application entry point during the engine bootstrap sequence. It is one of the first services to be initialized.
- **Scope:** The EventBus is a global singleton that persists for the entire lifetime of the application session. Its existence is tied directly to the running client or server process.
- **Destruction:** The EventBus is torn down during the final stages of application shutdown. This process typically involves clearing all listener registries and shutting down the associated thread pool for asynchronous events.

## Internal State & Concurrency
The IEventBus interface is stateless. However, any compliant implementation is inherently stateful and must be designed for high-concurrency environments.

- **State:** The underlying implementation maintains a complex, mutable state, primarily consisting of a map or similar data structure that associates event types (and keys) with collections of registered listeners. This registry is constantly modified at runtime as systems register and unregister listeners.
- **Thread Safety:** The contract demands that all implementations be fully thread-safe. Systems from any thread (e.g., main loop, networking, asset loading) must be able to dispatch events or register listeners without causing race conditions or deadlocks. Implementations typically achieve this through the use of concurrent collections and careful synchronization on the listener registry.

**WARNING:** While the bus itself is thread-safe, the responsibility for thread safety within an event listener's implementation lies with the developer. State modified by an asynchronous event handler must be protected by appropriate synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| dispatch(eventClass) | EventType | O(N) | Dispatches a synchronous event. Blocks until all listeners have executed. N is the number of listeners. |
| dispatchAsync(eventClass) | CompletableFuture | O(1) | Dispatches an asynchronous event to a background thread pool. Returns immediately. |
| dispatchFor(eventClass, key) | IEventDispatcher | O(1) | Creates a prepared dispatcher for a specific event type and optional key. This is a performance optimization. |
| dispatchForAsync(eventClass, key) | IEventDispatcher | O(1) | Creates a prepared dispatcher for an asynchronous event type and optional key. |

## Integration Patterns

### Standard Usage
The IEventBus should always be retrieved from the appropriate service context. It should never be instantiated directly. The primary interaction is to dispatch events for other systems to consume.

```java
// Example of a system dispatching a player join event
public class PlayerConnectionSystem {
    private final IEventBus eventBus;

    public PlayerConnectionSystem(ServiceContext context) {
        this.eventBus = context.getService(IEventBus.class);
    }

    public void onPlayerConnect(Player player) {
        // Create a prepared dispatcher for efficiency if dispatching often
        IEventDispatcher<PlayerJoinEvent, PlayerJoinEvent> dispatcher =
            eventBus.dispatchFor(PlayerJoinEvent.class, player.getId());

        // Dispatch the event
        dispatcher.dispatch(new PlayerJoinEvent(player));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Implementation:** Do not attempt to provide your own implementation of IEventBus unless you are performing a deep engine modification. The engine relies on a single, canonical instance.
- **Long-Running Synchronous Listeners:** Registering a listener for a synchronous event that performs a long-running or blocking operation (e.g., file I/O, complex pathfinding) is a critical performance error. This will freeze the dispatching thread, which is often the main game loop. Offload such work to an asynchronous event or a separate job.
- **Stateful Async Listeners:** Creating listeners for asynchronous events that modify shared, non-thread-safe state is a primary source of race conditions and data corruption. Any state touched by an async listener must be synchronized.

## Data Pipeline
The IEventBus acts as a central routing hub for in-memory data flow. It does not transform data but directs event objects from a single publisher to multiple subscribers.

> Flow:
> System A (Publisher) -> `IEventBus.dispatch(MyEvent)` -> **EventBus Core Logic (Listener Lookup)** -> [Listener 1, Listener 2, ...] -> System B, System C (Subscribers)

