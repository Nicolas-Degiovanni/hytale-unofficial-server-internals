---
description: Architectural reference for AssetStoreTypeHandler
---

# AssetStoreTypeHandler

**Package:** com.hypixel.hytale.builtin.asseteditor.assettypehandler
**Type:** Strategy / Transient

## Definition
```java
// Signature
public class AssetStoreTypeHandler extends JsonTypeHandler {
```

## Architecture & Concepts
The AssetStoreTypeHandler is a specialized implementation of the **AssetTypeHandler** contract. It acts as a critical bridge between the generic, file-oriented Asset Editor subsystem and a specific, concrete **AssetStore** instance within the game server.

In Hytale's architecture, an **AssetStore** is a high-performance, in-memory cache for a specific category of game assets, such as items, blocks, or prefabs. The Asset Editor, however, operates on a more abstract level, dealing with file paths and raw BSON data. This handler's primary function is to translate the generic "load", "unload", and "restore" commands from the Asset Editor into specific, state-changing operations on its designated **AssetStore**.

This class embodies the Strategy design pattern. The Asset Editor framework holds a collection of **AssetTypeHandler** strategies, and at runtime, it selects the correct one based on the path of the modified asset. For any asset managed by an **AssetStore**, this handler is the chosen strategy. It encapsulates the knowledge required to decode a BSON document into a game-ready asset object and correctly inject it into the live server state via the **AssetStore** API.

## Lifecycle & Ownership
-   **Creation:** An AssetStoreTypeHandler is not a singleton. A new instance is created for each concrete **AssetStore** that is registered with the Asset Editor system. This typically occurs during server bootstrap within the **AssetEditorPlugin**, which discovers all manageable asset stores and instantiates a corresponding handler for each.

-   **Scope:** The lifecycle of an AssetStoreTypeHandler is tightly coupled to its associated **AssetStore** and the **AssetEditorPlugin**. It persists for the entire duration of the server session in which the Asset Editor is active.

-   **Destruction:** The object is eligible for garbage collection when the server shuts down or the **AssetEditorPlugin** is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains two key pieces of internal state:
    1.  A final, non-null reference to the **AssetStore** it manages. This reference is immutable after construction.
    2.  A lazily-initialized **cachedDefaultUpdateQuery**. This field caches the result of a potentially expensive schema lookup, improving performance for subsequent requests.

-   **Thread Safety:** This class is **NOT THREAD-SAFE**. The lazy initialization of **cachedDefaultUpdateQuery** in the **getDefaultUpdateQuery** method involves a non-atomic check-then-act sequence (`if (this.cachedDefaultUpdateQuery == null)`). If multiple threads were to call this method concurrently, it could lead to a race condition where the schema is processed multiple times.

    **WARNING:** All interactions with an instance of this class must be externally synchronized or confined to a single thread, which is the standard operational model for the Asset Editor's file processing loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | Returns the underlying AssetStore instance this handler manages. |
| loadAssetFromDocument(...) | AssetLoadResult | O(N) | Decodes a BSON document into an asset and instructs the AssetStore to load it. N is the complexity of asset decoding and insertion. Propagates an AssetUpdateQuery. |
| unloadAsset(...) | AssetLoadResult | O(log N) | Instructs the AssetStore to remove an asset by its key. N is the number of assets in the store. |
| restoreOriginalAsset(...) | AssetLoadResult | O(N) | Instructs the AssetStore to reload an asset from its original pack data, discarding any live edits. N is the complexity of asset loading. |
| getDefaultUpdateQuery() | AssetUpdateQuery | O(N) / O(1) | Computes the default cache invalidation rules for this asset type. Complexity is O(N) on first call (schema lookup) and O(1) on subsequent calls due to caching. |

## Integration Patterns

### Standard Usage
A developer typically does not interact with this class directly. It is managed and invoked by the **AssetEditorPlugin** framework. The framework identifies the correct handler for a given file path and dispatches asset modification events to it.

```java
// Conceptual example of framework invocation
AssetPath changedAssetPath = ...;
BsonDocument document = ...;
AssetUpdateQuery updateQuery = new AssetUpdateQuery();

// The framework finds the correct handler for the path
AssetStoreTypeHandler handler = assetEditor.findHandlerFor(changedAssetPath);

// The framework invokes the handler to update the live game state
if (handler != null) {
    handler.loadAssetFromDocument(changedAssetPath, dataPath, document, updateQuery, editorClient);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AssetStoreTypeHandler()`. The framework is responsible for creating and registering handlers. Manual creation will result in a handler that is not known to the Asset Editor and will never be used.
-   **State Manipulation:** Do not attempt to modify the handler's underlying **AssetStore** from a different thread while the Asset Editor is processing an update. This will lead to concurrency issues and corrupted game state.
-   **Ignoring the Update Query:** The **AssetUpdateQuery** populated by methods like **loadAssetFromDocument** is critical for notifying clients to rebuild their caches. Failing to propagate this query will result in visual or gameplay desynchronization.

## Data Pipeline
The AssetStoreTypeHandler is a key component in the live asset reloading pipeline. It translates file system events into in-game state changes.

> Flow:
> File System Change -> AssetEditorPlugin -> **AssetStoreTypeHandler** -> AssetStore Update -> AssetUpdateQuery -> Network Packet -> Client Cache Rebuild

