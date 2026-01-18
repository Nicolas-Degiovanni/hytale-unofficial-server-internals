---
description: Architectural reference for RegisterAssetStoreEvent
---

# RegisterAssetStoreEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient Event Object

## Definition
```java
// Signature
public class RegisterAssetStoreEvent extends AssetStoreEvent<Void> {
```

## Architecture & Concepts
The RegisterAssetStoreEvent is a fundamental signaling mechanism within Hytale's asset management subsystem. It operates as a core component of an event-driven architecture, adhering to the Observer design pattern.

Its primary role is to decouple the creation of an AssetStore from its registration and management. When a new AssetStore is instantiated—for example, by a mod loader, a world loader, or a core engine system—it broadcasts this event to announce its existence. A central registry, such as a global AssetManager, listens for this event and integrates the new AssetStore into the engine's resource loading pipeline.

By using an event, the system avoids tight coupling and hard-coded dependencies, allowing for a modular and extensible asset framework where new sources of assets can be introduced dynamically without modifying the core manager. The use of Void as the generic type parameter from the parent AssetStoreEvent signifies that this is a one-way notification; listeners are not expected to return a result.

### Lifecycle & Ownership
- **Creation:** Instantiated by any system component that programmatically creates a new AssetStore instance. This is a common pattern for dynamic resource packs or mod initialization routines.
- **Scope:** Ephemeral. The event object's lifecycle is strictly limited to the duration of its dispatch on an event bus. It is created, posted, and becomes eligible for garbage collection immediately after all subscribers have processed it.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no explicit cleanup methods. Holding a reference to this event object after its dispatch is considered a memory leak.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal reference to the AssetStore is set once via the constructor and is not intended to be modified. The state is contained within the parent AssetStoreEvent class.
- **Thread Safety:** This class is inherently thread-safe due to its immutable nature. However, the context in which it is used requires careful consideration. The event may be posted from any thread. All subscribers (listeners) that consume this event **must** be designed for concurrent access and ensure that their handling logic, such as adding the new AssetStore to a collection, is properly synchronized.

## API Surface
The public contract is minimal, consisting of the constructor. The primary data accessor is inherited from the parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RegisterAssetStoreEvent(AssetStore) | constructor | O(1) | Creates a new registration event. Throws a NullPointerException if the provided AssetStore is null. |
| getAssetStore() | AssetStore | O(1) | Inherited from AssetStoreEvent. Returns the non-null AssetStore instance to be registered. |

## Integration Patterns

### Standard Usage
The correct pattern is to create an AssetStore and immediately post this event to the appropriate event bus, allowing interested systems to register it.

```java
// Example: A mod loader registering its own asset source
AssetStore modAssetStore = new FileSystemAssetStore(modDirectory);
EventBus eventBus = context.getService(EventBus.class);

// Announce the new AssetStore to the rest of the engine
eventBus.post(new RegisterAssetStoreEvent(modAssetStore));
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to use this event to communicate state changes about an *already registered* AssetStore. This event is strictly for initial registration. Use a different event, such as an AssetStoreReloadEvent, for such purposes.
- **Holding References:** Listeners must not maintain a reference to the RegisterAssetStoreEvent object itself. Extract the contained AssetStore and discard the event wrapper.
- **Synchronous Expectation:** Do not post this event and immediately expect the AssetStore to be available for asset loading. The registration process may be asynchronous depending on the event bus implementation and listener logic.

## Data Pipeline
This event acts as a data carrier in a simple, one-way pipeline that integrates new asset sources into the engine.

> Flow:
> System (e.g., Mod Loader) creates AssetStore -> **RegisterAssetStoreEvent** is instantiated -> Event is posted to EventBus -> Subscriber (e.g., AssetManager) receives event -> AssetManager adds the AssetStore to its internal registry.

