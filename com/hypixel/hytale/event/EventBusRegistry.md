---
description: Architectural reference for EventBusRegistry
---

# EventBusRegistry

**Package:** com.hypixel.hytale.event
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class EventBusRegistry<KeyType, EventType extends IBaseEvent<KeyType>, ConsumerMapType extends EventBusRegistry.EventConsumerMap<EventType, ?, ?>> {
```

## Architecture & Concepts

The EventBusRegistry is a foundational, high-performance component of the engine's event system. It is not a simple global event bus; rather, it is a sophisticated, abstract registry for managing multiple, distinct event channels that are partitioned by a generic KeyType. This design allows for fine-grained event routing, which is critical for performance in a complex entity-component system or a multi-threaded networking environment.

The core architectural pattern is the separation of event listeners into three distinct scopes:

1.  **Keyed Listeners:** The primary and most common type. These listeners are registered against a specific key (e.g., an entity ID, a network connection ID, a UI widget name). When an event is dispatched with a key, only the listeners registered for that exact key are notified. This avoids costly iteration over thousands of unrelated listeners.
2.  **Global Listeners:** These listeners are invoked for *every* event of the registered EventType, regardless of its key. This is suitable for cross-cutting concerns like logging, metrics, or global state management.
3.  **Unhandled Listeners:** A fallback mechanism. These listeners are invoked only when an event is dispatched for a key that has no specific keyed listeners registered. This is useful for debugging or handling default cases.

The system is engineered for extreme concurrency and low-latency dispatch. It achieves this through careful selection of thread-safe data structures and non-blocking algorithms, making it safe for use across high-contention threads like the main game loop, physics, and networking threads.

### Lifecycle & Ownership
-   **Creation:** A concrete implementation of EventBusRegistry is instantiated once by a primary service container or engine entry point during the application bootstrap sequence. It is a foundational service, expected to be available very early in the engine's startup process.
-   **Scope:** The registry is a long-lived object. Its lifecycle is tied to the entire application session, persisting from client/server start to shutdown.
-   **Destruction:** The shutdown method must be called explicitly during the engine's shutdown sequence. This action clears all internal listener maps, preventing memory leaks and ensuring that no further events can be processed on the terminated bus. Failure to call shutdown can lead to resource leaks.

## Internal State & Concurrency
-   **State:** The EventBusRegistry is highly mutable. Its internal state consists of several maps that store collections of event consumers. These maps are continuously modified at runtime as various engine systems register and unregister listeners.
-   **Thread Safety:** This class is designed to be fully thread-safe.
    -   The primary map of keyed listeners is a ConcurrentHashMap, allowing for safe concurrent registration from different threads.
    -   Within each EventConsumerMap, listeners are stored in a CopyOnWriteArrayList. This data structure provides extremely fast, lock-free iteration, which is the most frequent operation (event dispatch). Registration and unregistration operations are more costly as they require creating a new copy of the underlying array, but these are considered infrequent.
    -   The sorted array of listener priorities is managed by an AtomicReference and updated using a non-blocking compare-and-set (CAS) loop. This ensures that the priority list remains consistent and sorted even under heavy concurrent modification.

## API Surface

The public contract is defined by abstract methods that must be implemented by a concrete subclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(priority, key, consumer) | EventRegistration | O(log P + C) | **Abstract.** Registers a key-specific listener. Complexity depends on priority list size (P) and copy cost of consumer list (C). |
| registerGlobal(priority, consumer) | EventRegistration | O(log P + C) | **Abstract.** Registers a listener that receives all events of this type. |
| registerUnhandled(priority, consumer) | EventRegistration | O(log P + C) | **Abstract.** Registers a listener for events with no keyed listeners. |
| dispatchFor(key) | IEventDispatcher | O(1) | **Abstract.** Retrieves a dispatcher object for a specific key to begin the event firing process. |
| shutdown() | void | O(N) | Deactivates the registry and clears all N registered listeners. Any subsequent calls will fail or be ignored. |
| isAlive() | boolean | O(1) | Returns false if shutdown has been called. Essential for graceful system teardown. |

## Integration Patterns

### Standard Usage

A system should retrieve a concrete registry instance from the central service context. It should never create its own. Listeners are then registered, and the returned EventRegistration object can be stored to unregister later if necessary.

```java
// How a developer should normally use this
// Assume MyEventBusRegistry is a concrete implementation
MyEventBusRegistry eventBus = context.getService(MyEventBusRegistry.class);
EntityID targetEntity = ...;

// Register a listener for a specific entity
EventRegistration<EntityID, EntityDamagedEvent> registration = eventBus.register(
    EventPriority.NORMAL,
    targetEntity,
    event -> {
        System.out.println("Entity " + event.getKey() + " took damage.");
    }
);

// ... later, when the system is shutting down
registration.unregister();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MyEventBusRegistry()`. The registry is a managed singleton service. Always retrieve it from the application's service locator or dependency injection container.
-   **Ignoring Shutdown State:** Do not attempt to register listeners or dispatch events on a registry after `shutdown()` has been called. Always check `isAlive()` during system teardown phases to avoid race conditions.
-   **Leaked Registrations:** Do not forget to unregister listeners when the owning object is destroyed. Forgetting to call `registration.unregister()` will result in a memory leak and cause "ghost" callbacks from dead objects.

## Data Pipeline

The EventBusRegistry acts as a central routing hub. The flow for a dispatched event is governed by its key and the presence of registered listeners across the three scopes.

> Flow:
> Event Source -> `registry.dispatchFor(event.getKey())` -> **EventBusRegistry** (Key-to-ConsumerMap Lookup) -> `EventConsumerMap` (Priority-Sorted Iteration) -> Invocation of registered `Consumer<EventType>` lambdas.

