---
description: Architectural reference for RemovedAssetsEvent
---

# RemovedAssetsEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient

## Definition
```java
// Signature
public class RemovedAssetsEvent<K, T extends JsonAsset<K>, M extends AssetMap<K, T>> extends AssetsEvent<K, T> {
```

## Architecture & Concepts
The RemovedAssetsEvent is an immutable data-transfer object that functions as a core message within the engine's event-driven asset management system. It represents a discrete, historical fact: one or more assets have been unloaded from memory.

This class is not a service or manager; it is a notification. Its primary architectural role is to decouple the AssetStore, which manages the loading and unloading of game assets, from the various engine subsystems that consume those assets. When the AssetStore purges assets from an AssetMap, it broadcasts a RemovedAssetsEvent. Subsystems like the renderer, audio engine, or entity manager can listen for this event to safely deallocate associated resources and update their internal state without being tightly coupled to the AssetStore's implementation.

A key architectural feature is the **replaced** flag. This allows listeners to differentiate between a permanent asset removal (e.g., leaving a game zone) and a temporary removal that is part of a hot-reload or update cycle. This prevents expensive deallocation and reallocation of resources when an asset is merely being updated.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the AssetStore or its internal delegates when assets are successfully removed from an AssetMap. Client systems should never create instances of this event.
- **Scope:** Ephemeral. An instance of RemovedAssetsEvent exists only for the duration of its dispatch through the event bus. It is a short-lived messenger object.
- **Destruction:** Once all registered event listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. The object's state is fully defined at construction and cannot be changed thereafter. The internal set of removed asset keys is explicitly wrapped in an unmodifiable collection to enforce this contract at runtime.
- **Thread Safety:** Inherently **Thread-Safe**. Due to its immutability, an instance of RemovedAssetsEvent can be safely published from a background asset-loading thread and consumed by listeners on the main game thread without any need for locks or other synchronization primitives.

## API Surface
The public API consists entirely of non-blocking, O(1) accessor methods for retrieving the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetClass() | Class<T> | O(1) | Returns the class token for the type of asset that was removed. |
| getAssetMap() | M | O(1) | Provides a reference to the AssetMap instance from which the assets were removed. |
| getRemovedAssets() | Set<K> | O(1) | Returns an unmodifiable set of keys corresponding to the removed assets. |
| isReplaced() | boolean | O(1) | Returns true if the removal is part of a hot-reload operation. |

## Integration Patterns

### Standard Usage
The canonical use case is to implement an event listener that subscribes to this event type. The listener logic inspects the event payload to determine if action is required.

```java
// A renderer system listening for texture removals
@Subscribe
public void onAssetsRemoved(RemovedAssetsEvent event) {
    // Only act on events related to Texture assets
    if (event.getAssetClass() == Texture.class) {
        // If it's a hot-reload, we might defer GPU cleanup
        if (event.isReplaced()) {
            // Log or schedule a delayed cleanup
            return;
        }

        // For permanent removals, purge GPU resources
        for (Object key : event.getRemovedAssets()) {
            gpuTextureCache.evict((TextureKey) key);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new RemovedAssetsEvent()`. This event is a notification *from* the asset system, not a request *to* it. Creating your own instance will have no effect on the engine state.
- **Attempting State Mutation:** The set returned by getRemovedAssets is unmodifiable. Attempting to add or remove elements from it will result in an UnsupportedOperationException at runtime.
- **Ignoring the isReplaced Flag:** Failing to check `isReplaced()` can lead to inefficient resource handling. A system might deallocate a GPU texture, only to have to immediately re-upload it when the corresponding AddedAssetsEvent for the new version arrives nanoseconds later.

## Data Pipeline
The RemovedAssetsEvent is a critical component in the asset unloading data flow. It serves as the formal notification that signals the completion of an unload operation and triggers downstream cleanup tasks.

> Flow:
> Unload Trigger (e.g., Zone Unload, Hot-Reload) -> AssetStore Unload Logic -> **RemovedAssetsEvent** Instantiation -> Event Bus Dispatch -> Subsystem Listeners (Renderer, Audio, etc.) -> System-Specific Resource Deallocation (e.g., GPU memory free)

