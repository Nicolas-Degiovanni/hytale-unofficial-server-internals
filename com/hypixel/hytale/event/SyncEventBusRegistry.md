---
description: Architectural reference for SyncEventBusRegistry
---

# SyncEventBusRegistry

**Package:** com.hypixel.hytale.event
**Type:** Transient (Managed as a Singleton Service)

## Definition
```java
// Signature
public class SyncEventBusRegistry<KeyType, EventType extends IEvent<KeyType>>
   extends EventBusRegistry<KeyType, EventType, SyncEventBusRegistry.SyncEventConsumerMap<EventType>> {
```

## Architecture & Concepts

The SyncEventBusRegistry is a foundational component of the engine's event handling system, providing a mechanism for synchronous, single-threaded, and priority-ordered event dispatch. It acts as a central hub for a specific class of events, identified by its generic `EventType`.

Its primary architectural function is to decouple event producers from event consumers. Instead of direct method calls, a system can create an event object and submit it to the registry, which then routes it to all interested and registered listeners.

The system is designed around three distinct listener scopes:

1.  **Keyed Listeners:** These are the most common type. Listeners register their interest in events associated with a specific `KeyType`. This allows for highly targeted event delivery. For example, an `EntityDamageEvent` might be keyed by the entity's unique ID, ensuring that only systems concerned with that specific entity are notified.
2.  **Global Listeners:** These listeners receive *every* event of the registered `EventType`, regardless of its key. This is useful for global systems like logging, analytics, or systems that need a comprehensive view of all events of a certain type.
3.  **Unhandled Listeners:** This is a fallback mechanism. If an event is dispatched for a key that has *no* registered keyed listeners, it is routed to the unhandled listeners. This prevents events from being silently dropped and allows for default handling logic.

Dispatch order is strictly defined: an event first attempts to dispatch to keyed listeners. It is then *always* dispatched to global listeners. If and only if there were no keyed listeners, it is dispatched to unhandled listeners. Within each scope, listeners are executed according to a signed short priority value, from lowest to highest.

## Lifecycle & Ownership

-   **Creation:** SyncEventBusRegistry instances are not intended for direct instantiation by feature developers. They are created and managed by a higher-level service container or the central `EventBus`. Typically, one instance is created on-demand for each unique `EventType` that requires a bus.
-   **Scope:** An instance persists for the lifetime of a major engine context, such as a client session or a game server instance. It is not re-created during scene loads or world changes.
-   **Destruction:** The registry is terminated when its parent context is destroyed. A `shutdown` flag is set, which causes all subsequent attempts to register, unregister, or dispatch events to throw an `IllegalArgumentException`. This is a hard failure designed to prevent use-after-free errors.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable, consisting of several maps that store collections of event consumers. The primary state is the map of keys to their corresponding listener lists. This state is modified exclusively through the public `register` and `unregister` methods. The registry does not cache or store event objects themselves.

-   **Thread Safety:** **WARNING:** This class is **NOT THREAD-SAFE** and is designed for single-threaded access. The "Sync" in its name refers to the synchronous execution of listeners, not to thread synchronization primitives. All registration, unregistration, and dispatch operations for a given registry instance **MUST** be performed on the same thread, typically the main engine update thread. Failure to adhere to this will result in `ConcurrentModificationException` or other catastrophic, non-deterministic behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(priority, key, consumer) | EventRegistration | O(log P) | Registers a key-specific listener. Returns a handle for unregistration. |
| registerGlobal(priority, consumer) | EventRegistration | O(log P) | Registers a listener that receives all events of this type. |
| registerUnhandled(priority, consumer) | EventRegistration | O(log P) | Registers a listener for events with no key-specific listeners. |
| dispatchFor(key) | IEventDispatcher | O(1) | Retrieves a dispatcher for a given key. This is the primary entry point for firing events. |

*Complexity Note: P refers to the number of distinct priority levels.*

## Integration Patterns

### Standard Usage

The standard pattern involves retrieving the registry for a specific event type from a central service, registering listeners during system initialization, and using the `dispatchFor` method to fire events during runtime.

```java
// During system initialization
// Assume 'eventBus' is a central service that manages registries
SyncEventBusRegistry<EntityId, EntityDamageEvent> damageRegistry = eventBus.getRegistry(EntityDamageEvent.class);

// Register a high-priority listener for a specific entity
damageRegistry.register(Priority.HIGH, myEntity.getId(), event -> {
    // Apply armor reduction logic
});

// During the game loop
EntityDamageEvent damageEvent = new EntityDamageEvent(myEntity.getId(), 10.0);
IEventDispatcher<EntityDamageEvent, EntityDamageEvent> dispatcher = damageRegistry.dispatchFor(myEntity.getId());
dispatcher.dispatch(damageEvent);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SyncEventBusRegistry()`. The engine manages the lifecycle of these objects. Always retrieve a shared instance from the central `EventBus` or service container. Creating your own instance will result in a disconnected event bus that receives no engine events.
-   **Cross-Thread Access:** Do not register a listener from one thread and dispatch an event from another. This will corrupt the internal state of the listener maps and lead to crashes. All interactions must be synchronized to the main game thread.
-   **Storing Dispatchers:** Do not retrieve and store the result of `dispatchFor` in a long-lived field. The registry can return an optimized `NO_OP` dispatcher if no listeners exist. This can change as listeners are added or removed. Always call `dispatchFor` immediately before dispatching an event to get the correct dispatcher for the current state.

## Data Pipeline

The flow of an event through the registry is a control flow, not a data transformation pipeline. The event object itself is passed by reference to all consumers.

> Flow:
> Event Source -> `dispatchFor(key)` -> `IEventDispatcher.dispatch(event)` -> **SyncEventBusRegistry**
>
> **Internal Routing:**
> 1.  Route to Keyed Listeners (if any exist for the key)
> 2.  Route to all Global Listeners
> 3.  Route to Unhandled Listeners (only if no Keyed Listeners were found in step 1)
>
> -> Event Handled by Consumers

