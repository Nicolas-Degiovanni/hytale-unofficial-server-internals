---
description: Architectural reference for LoadedAssetsEvent
---

# LoadedAssetsEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient

## Definition
```java
// Signature
public class LoadedAssetsEvent<K, T extends JsonAsset<K>, M extends AssetMap<K, T>> extends AssetsEvent<K, T> {
```

## Architecture & Concepts
The LoadedAssetsEvent is a fundamental signaling mechanism within the Hytale Asset Pipeline. It is not a service or manager, but rather an immutable data-transfer object that represents a completed asset loading operation.

Its primary role is to decouple the AssetStore, which is responsible for I/O and asset hydration, from the various engine systems that consume those assets. When the AssetStore fulfills an AssetUpdateQuery, it broadcasts a LoadedAssetsEvent to announce the availability of new data.

This event-driven approach allows systems like the RenderSystem, AudioEngine, or a UI framework to react to newly available assets without polling or maintaining a direct dependency on the AssetStore's internal state. The event carries a complete, self-contained snapshot of the load operation, including the specific assets loaded, the query that triggered it, and the target AssetMap that was updated.

### Lifecycle & Ownership
- **Creation:** An instance is created exclusively by the AssetStore or its internal worker threads. This occurs at the final stage of an asset loading task, after assets have been successfully read from a source and deserialized into memory.
- **Scope:** Extremely short-lived. The object's lifecycle is bound to its dispatch on the engine's central event bus. It is created, broadcast to all subscribers, and then becomes eligible for garbage collection. It is not intended to be stored or cached by any system.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** **Immutable**. All fields are final and set only once during construction. Critically, the map of loaded assets is wrapped in an unmodifiable view (`Collections.unmodifiableMap`), which prevents any downstream consumer from altering the event's payload. This guarantees that all subscribers receive the exact same, consistent data.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a LoadedAssetsEvent instance can be safely passed across thread boundaries without any need for locks or synchronization. An asset loading worker thread can create and dispatch the event, and it can be consumed on the main game thread without risk of race conditions or data corruption.

## API Surface
The public API is composed entirely of non-mutating accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetClass() | Class<T> | O(1) | Returns the runtime class of the assets contained in this event. |
| getAssetMap() | M | O(1) | Returns a reference to the master AssetMap that was updated by this load operation. |
| getLoadedAssets() | Map<K, T> | O(1) | Returns a read-only map of the specific assets that were just loaded. |
| isInitial() | boolean | O(1) | Returns true if this event represents the first-ever load for this asset type. |
| getQuery() | AssetUpdateQuery | O(1) | Returns the original query object that initiated the asset loading process. |

## Integration Patterns

### Standard Usage
The only correct way to interact with this class is to subscribe to it through the engine's event bus. Systems register a listener and filter events based on the asset type they are interested in.

```java
// A system subscribes to the event to receive newly loaded textures.
eventBus.subscribe(LoadedAssetsEvent.class, (event) -> {
    // WARNING: Type checking is essential as events for all assets are broadcast.
    if (event.getAssetClass() == Texture.class) {
        Map<String, Texture> newTextures = event.getLoadedAssets();
        
        // The isInitial flag is critical for one-time setup logic.
        if (event.isInitial()) {
            this.initializeTextureAtlas(newTextures);
        } else {
            this.hotReloadTextures(newTextures);
        }
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of LoadedAssetsEvent manually using `new`. Doing so would constitute a fake signal and would lead to severe state desynchronization, as no assets were actually loaded into the backing AssetMap. These events are the exclusive responsibility of the AssetStore.
- **Attempting to Modify Payload:** The map returned by getLoadedAssets is unmodifiable. Any attempt to call methods like `put`, `remove`, or `clear` will result in an `UnsupportedOperationException`. Treat the event as a read-only report.
- **Ignoring isInitial Flag:** Systems that perform expensive, one-time initialization logic must check the `isInitial()` flag. Failing to do so will cause this logic to re-run incorrectly during subsequent hot-reloads or dynamic asset streaming, potentially causing bugs or performance degradation.

## Data Pipeline
The LoadedAssetsEvent is the final output of the core asset loading pipeline. It serves as the bridge between the asynchronous I/O layer and the synchronous game simulation layer.

> Flow:
> AssetUpdateQuery -> AssetStore -> Asset Loader (Disk/Network I/O) -> **LoadedAssetsEvent** -> Event Bus -> Subscribing Systems (e.g., RenderSystem, SoundSystem, UI)

