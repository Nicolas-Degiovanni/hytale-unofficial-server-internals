---
description: Architectural reference for IEventRegistry
---

# IEventRegistry

**Package:** com.hypixel.hytale.event
**Type:** Interface / Service Contract

## Definition
```java
// Signature
public interface IEventRegistry {
```

## Architecture & Concepts
The IEventRegistry interface defines the contract for the engine's core event bus. It is the primary mechanism for decoupled, system-to-system communication, implementing a sophisticated Observer (Pub/Sub) pattern. Its design allows disparate parts of the engine, such as the physics system, UI layer, and network protocol handler, to communicate without holding direct references to one another.

This architecture is built on several key principles:

*   **Type-Safe Subscriptions:** Listeners subscribe to specific event classes. The registry guarantees that a listener for a PlayerJoinEvent will only ever receive objects of that type, enforced at compile time.

*   **Keyed Event Dispatch:** The use of a generic KeyType on IBaseEvent allows for highly targeted event delivery. Instead of a single, global PlayerDamageEvent that is broadcast to all listeners, the event can be keyed to a specific EntityID. The registry uses this key to invoke only the listeners that have registered for that specific entity, dramatically improving performance by avoiding unnecessary listener invocations.

*   **Prioritized Execution:** Listeners can be registered with an EventPriority or a raw short value. When an event is dispatched, the registry sorts all relevant listeners by this priority before invocation. This creates a predictable and deterministic execution order, which is critical for systems that rely on sequential processing, such as one system cancelling an event before another system processes it.

*   **Asynchronous Processing:** The interface provides a clear distinction between synchronous (register) and asynchronous (registerAsync) listeners. Asynchronous handlers receive and return a CompletableFuture, allowing long-running tasks like file I/O or complex calculations to be performed off the main game thread, preventing stalls and maintaining application responsiveness.

*   **Global and Unhandled Hooks:** The registry supports global listeners (registerGlobal) that receive all instances of an event, regardless of key. It also provides a mechanism to handle events for which no primary listener was found (registerUnhandled), which is invaluable for debugging, logging, and implementing default behaviors.

## Lifecycle & Ownership
As an interface, IEventRegistry has no lifecycle itself. However, its concrete implementation within the engine follows a strict singleton pattern scoped to a major application context (e.g., Client, Server).

*   **Creation:** A single instance of the event registry implementation is created by the top-level application container (e.g., ClientEntryPoint or ServerEntryPoint) during the engine's initial bootstrap sequence. It is one of the first foundational services to be initialized.
*   **Scope:** The service instance persists for the entire lifetime of the application process. It is a session-scoped singleton.
*   **Destruction:** The registry is torn down during the final stages of application shutdown. A compliant implementation is responsible for clearing all listener collections to ensure proper garbage collection and prevent memory leaks from stale object references.

## Internal State & Concurrency
Any implementation of IEventRegistry is, by definition, a highly stateful and contended resource.

*   **State:** The internal state consists of a complex mapping of event types and keys to collections of listener registrations. This state is highly mutable, as listeners are frequently added and removed throughout the application's lifecycle.

*   **Thread Safety:** Implementations **must be thread-safe**. Listeners can be registered from any thread, and events can be fired from the main game loop, network threads, or worker threads simultaneously. A robust implementation is expected to use concurrent data structures or explicit read-write locks to manage access to the internal listener maps, preventing race conditions and ensuring stable event dispatch. Firing an event is a read-heavy operation, while registering or unregistering is a write operation.

## API Surface
The public contract is extensive, with numerous overloads for convenience. The methods can be grouped into four primary categories of registration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(...) | EventRegistration | O(1) | Registers a synchronous, keyed listener. The handler is a Consumer. |
| registerAsync(...) | EventRegistration | O(1) | Registers an asynchronous, keyed listener. The handler is a Function that transforms a CompletableFuture. |
| registerGlobal(...) | EventRegistration | O(1) | Registers a listener that receives all instances of an event, ignoring any keys. |
| registerUnhandled(...) | EventRegistration | O(1) | Registers a fallback listener that is only invoked if an event has no other registered handlers. |

**Warning:** The returned EventRegistration object is the handle to the subscription. It is the caller's responsibility to store this handle and use it to unregister the listener when it is no longer needed. Failure to do so will result in memory leaks.

## Integration Patterns

### Standard Usage
Systems should retrieve the IEventRegistry instance from the central service context. Never attempt to create your own instance.

```java
// Example: A service listening for a player connecting
public class PlayerService {
    private final IEventRegistry eventRegistry;
    private EventRegistration<UUID, PlayerConnectEvent> registration;

    public PlayerService(EngineContext context) {
        this.eventRegistry = context.getService(IEventRegistry.class);
    }

    public void onInit() {
        // Listen for all PlayerConnectEvent events
        this.registration = eventRegistry.registerGlobal(PlayerConnectEvent.class, this::handlePlayerConnect);
    }

    private void handlePlayerConnect(PlayerConnectEvent event) {
        // Logic to handle player connection
    }

    public void onShutdown() {
        // CRITICAL: Unregister the listener to prevent leaks
        if (this.registration != null) {
            this.registration.unregister();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
*   **Long-Running Synchronous Listeners:** Never perform blocking I/O, network requests, or heavy computation inside a standard (synchronous) event listener. This will block the calling thread, which is often the main game thread, causing severe performance degradation or a complete application freeze. Use registerAsync for such operations.
*   **Ignoring the EventRegistration Handle:** Failing to store the returned EventRegistration object makes it impossible to unregister the listener. This is a common source of memory leaks, especially for listeners tied to temporary objects like UI elements or in-game entities.
*   **Modifying Collections During Dispatch:** Do not attempt to register or unregister listeners for an event type from within a listener for that same event type. While some implementations may be robust enough to handle this, it can lead to unpredictable behavior, such as ConcurrentModificationException or missed event notifications.

## Data Pipeline
The flow of a single event through the registry is a multi-stage process designed for performance and predictability.

> Flow:
> Event Source (e.g., Network Handler) -> `EventBus.fire(event)` -> **IEventRegistry Implementation** -> Listener Lookup (by Event Class and Key) -> Priority Sorting of Listeners -> Synchronous/Asynchronous Invocation Chain -> (Optional) Unhandled Listener Invocation

