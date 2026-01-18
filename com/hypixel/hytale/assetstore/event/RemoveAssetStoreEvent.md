---
description: Architectural reference for RemoveAssetStoreEvent
---

# RemoveAssetStoreEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient

## Definition
```java
// Signature
public class RemoveAssetStoreEvent extends AssetStoreEvent<Void> {
```

## Architecture & Concepts
The RemoveAssetStoreEvent is a message-based signal within the engine's event-driven architecture. It serves a single, critical purpose: to notify the system that a specific AssetStore instance is being decommissioned and should be removed from active management.

This class is not a service or a manager; it is an immutable data carrier. Its existence represents a command to be processed by one or more listener systems, typically a central AssetStoreRegistry. The use of the generic type Void in its parent class, AssetStoreEvent<Void>, explicitly indicates that this is a one-way notification. Systems that handle this event are not expected to return a value or result. The event's sole responsibility is to carry the identity of the AssetStore targeted for removal.

## Lifecycle & Ownership
- **Creation:** Instantiated by high-level state managers when a major system boundary is crossed. Common triggers include unloading a world, disconnecting from a server, or disabling a mod, all of which necessitate the teardown of their associated asset containers.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the central EventBus. It is created, posted, and becomes eligible for garbage collection immediately after all subscribers have processed it.
- **Destruction:** The Java Garbage Collector reclaims the memory for the event object once the EventBus dispatch cycle completes and no further strong references exist.

## Internal State & Concurrency
- **State:** Immutable. The target AssetStore is provided via the constructor and stored in a final field within the parent class. Once constructed, the event's state cannot be altered. This immutability is essential for predictable behavior in a multi-threaded environment.
- **Thread Safety:** This class is inherently thread-safe. As an immutable data object, it can be safely passed between threads without locks or synchronization.

    **WARNING:** While the event object itself is thread-safe, the listeners that consume it are responsible for ensuring their own thread safety. A common listener, the AssetStoreRegistry, must use appropriate synchronization when modifying its internal collection of active AssetStore instances in response to this event.

## API Surface
The public contract is minimal, consisting only of the constructor. Data is retrieved via inherited methods from the parent AssetStoreEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RemoveAssetStoreEvent(assetStore) | Constructor | O(1) | Constructs a new event targeting the specified AssetStore for removal. Throws NullPointerException if assetStore is null. |

## Integration Patterns

### Standard Usage
This event should never be handled directly. It must be posted to the global or context-specific EventBus, allowing the appropriate systems to react to it declaratively.

```java
// Example: A WorldManager cleaning up after a world is unloaded
AssetStore worldAssetStore = world.getAssetStore();
EventBus eventBus = context.getService(EventBus.class);

// Post the event to the bus for system-wide handling
eventBus.post(new RemoveAssetStoreEvent(worldAssetStore));
```

### Anti-Patterns (Do NOT do this)
- **Retaining References:** Do not store an instance of this event in a field or long-lived collection. Its purpose is fulfilled the moment it is posted to the EventBus. Retaining it can prevent garbage collection and implies incorrect lifecycle management.
- **Direct Listener Invocation:** Never create an instance of this event and pass it directly to a listener method. This bypasses the EventBus and breaks the decoupled, publish-subscribe architecture, risking that other critical systems will not be notified of the removal.

## Data Pipeline
The flow of this event is linear and unidirectional, acting as a trigger for a cleanup process.

> Flow:
> High-Level System Trigger (e.g., World Unload) -> Manager creates **RemoveAssetStoreEvent** -> EventBus Dispatch -> AssetStoreRegistry Listener -> Target AssetStore is deregistered and its resources are queued for release.

