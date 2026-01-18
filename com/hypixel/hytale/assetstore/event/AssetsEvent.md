---
description: Architectural reference for AssetsEvent
---

# AssetsEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Base Class

## Definition
```java
// Signature
public abstract class AssetsEvent<K, T extends JsonAsset<K>> implements IEvent<Class<T>> {
```

## Architecture & Concepts
AssetsEvent is an abstract base class that forms the foundation of the event-driven architecture within the Hytale Asset Store subsystem. It establishes a standardized, type-safe contract for all events related to the lifecycle of `JsonAsset` objects.

Its primary architectural role is to decouple the `AssetManager` from the various engine systems that consume assets. Instead of direct, high-coupling calls from the asset loader to systems like the renderer or audio engine, the `AssetManager` dispatches concrete subclasses of AssetsEvent onto a central event bus. Systems subscribe to these events to react to changes in asset state, such as loading completion or hot-reloading.

The use of generics is critical. `K` represents the asset's key type (e.g., a String identifier), and `T` represents the asset type itself. The implementation of `IEvent<Class<T>>` is a deliberate design choice: listeners subscribe to events concerning an entire *class* of assets (e.g., `Texture.class`) rather than a single asset instance. This allows for broad, category-based reactions, simplifying system logic.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of AssetsEvent are instantiated by the core asset management systems, typically in response to an internal state change. For example, an `AssetsLoadedEvent` would be created by the `AssetManager` after a batch of assets has been successfully loaded into memory.
- **Scope:** The lifecycle of an AssetsEvent instance is extremely brief and transient. It exists only for the duration of its dispatch and processing by the event bus. Once all listeners have handled the event, it is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Automatic. There is no manual cleanup required. The object is designed to be a short-lived message.

## Internal State & Concurrency
- **State:** Immutable. As a base class, AssetsEvent defines no state. Subclasses are expected to follow this pattern, acting as immutable Data Transfer Objects (DTOs). They should be populated with all necessary data at construction time and never modified thereafter.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, an AssetsEvent object can be safely passed between threads without locks or synchronization. The responsibility for thread-safe dispatch and listener execution lies with the event bus implementation, not the event object itself.

## API Surface
As an abstract class, AssetsEvent has no direct public API. It enforces a contract inherited from the `IEvent` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSubject() | Class<T> | O(1) | (Inherited from IEvent) Returns the asset class this event pertains to. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or use AssetsEvent directly. The primary interaction is to create a concrete subclass to represent a specific domain event or to create a listener that subscribes to such an event.

```java
// 1. Defining a concrete event
public class TextureAssetsLoaded extends AssetsEvent<String, Texture> {
    // Constructor and data fields...
}

// 2. Listening for the event in a system
@Subscribe
public void onTextureAssetsLoaded(TextureAssetsLoaded event) {
    // Logic to refresh rendering components that use textures
    log.info("Texture assets have been loaded. Subject: " + event.getSubject().getName());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate `new AssetsEvent<>()`. It is an abstract class and will result in a compilation error.
- **Mutable Subclasses:** Do not design concrete subclasses with public setters or mutable fields. An event must be a stable, immutable record of something that has already occurred. Modifying an event after dispatch leads to unpredictable behavior and breaks the contract.
- **Overly Broad Listeners:** Avoid listening for the raw `AssetsEvent.class`. This is almost always incorrect, as it provides no specific context. Listen for concrete, meaningful subclasses like `AssetsLoadedEvent` or `AssetReloadedEvent`.

## Data Pipeline
The flow of data for an asset-related event is unidirectional and follows the standard publish-subscribe pattern.

> Flow:
> AssetManager State Change -> **new ConcreteAssetsEvent()** -> EventBus.dispatch() -> System Listeners -> Engine State Update

