---
description: Architectural reference for AssetStoreEvent
---

# AssetStoreEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient Data Transfer Object (Base Class)

## Definition
```java
// Signature
public abstract class AssetStoreEvent<KeyType> implements IEvent<KeyType> {
```

## Architecture & Concepts
The AssetStoreEvent class is an abstract base for all events originating from the AssetStore subsystem. It is not a concrete, usable event but rather a foundational contract that guarantees any event extending it will carry a reference to its originating AssetStore.

This design is critical for a decoupled architecture. Systems subscribing to asset-related events do not need a direct dependency on a specific AssetStore instance. Instead, they can receive an event, inspect its payload, and use the `getAssetStore` method to identify the source. This allows for multiple, distinct AssetStore instances (e.g., for textures, audio, and models) to publish events on the same bus without ambiguity.

By implementing the IEvent interface, AssetStoreEvent signals its role as a payload carrier within the engine's primary event bus system.

### Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses, such as AssetLoadedEvent or AssetFailureEvent, are instantiated by an AssetStore instance when a relevant internal state change occurs, such as the completion or failure of an asset loading operation.
- **Scope:** Extremely short-lived. An AssetStoreEvent's lifecycle is confined to its transit through the event bus. It is created, published, consumed by all relevant subscribers, and then becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. It is typically dereferenced and destroyed immediately after the event dispatch cycle completes, assuming no subscribers retain a reference to it.

## Internal State & Concurrency
- **State:** Immutable. The sole piece of state, the reference to the source AssetStore, is a final field initialized at construction. This immutability is a deliberate design choice to ensure that the event's payload cannot be mutated during its dispatch across different threads.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, an AssetStoreEvent instance can be safely passed from a background worker thread (e.g., an asset loading thread) to the main game thread without requiring any locks or synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on providing context about the event's origin.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore<?, ?, ?> | O(1) | Returns the non-null AssetStore instance that published this event. |

## Integration Patterns

### Standard Usage
Developers will not interact with this class directly. Instead, they will write subscribers for concrete event types that extend AssetStoreEvent. The base class methods are then used to retrieve context.

```java
// A listener method in a system that needs to react to asset loading
@Subscribe
public void onAssetReady(AssetLoadedEvent event) {
    // Use the method from the AssetStoreEvent base class to get context
    AssetStore sourceStore = event.getAssetStore();
    log.info("Asset loaded from store: " + sourceStore.getName());

    // Process the specific payload from the concrete event
    Asset loadedAsset = event.getPayload();
    // ... update game state with the new asset
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to instantiate this class using `new AssetStoreEvent()` because it is abstract. This is by design.
- **Retaining References:** Do not store event objects in long-lived collections or as member variables. Events are transient and holding references to them can prevent garbage collection, leading to memory leaks. Process the event and then discard it.

## Data Pipeline
The AssetStoreEvent serves as a standardized data carrier in the flow from the asset subsystem to the rest of the engine.

> Flow:
> AssetStore Operation -> Concrete Event Instantiation -> EventBus.post() -> **AssetStoreEvent** (as payload) -> Event Subscriber -> Game State Update

