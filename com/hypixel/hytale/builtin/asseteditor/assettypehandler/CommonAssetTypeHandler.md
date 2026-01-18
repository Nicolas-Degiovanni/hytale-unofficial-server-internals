---
description: Architectural reference for CommonAssetTypeHandler
---

# CommonAssetTypeHandler

**Package:** com.hypixel.hytale.builtin.asseteditor.assettypehandler
**Type:** Transient

## Definition
```java
// Signature
public class CommonAssetTypeHandler extends AssetTypeHandler {
```

## Architecture & Concepts
The CommonAssetTypeHandler is a concrete implementation of the **Strategy** pattern, specializing in the management of "common" assets within the Hytale Asset Editor framework. Common assets are typically client-side resources like textures, models, or configuration files that are shared among all players and are not tied to dynamic game logic.

This class serves as a critical intermediary between high-level Asset Editor commands and the low-level server asset infrastructure. Its primary responsibility is to translate abstract asset modification requests (load, unload, restore) into concrete operations against the static `CommonAssetRegistry`.

It integrates tightly with two key singletons:
1.  **CommonAssetRegistry:** The canonical, in-memory store for all loaded common assets on the server. This handler is the designated component for mutating the state of this registry.
2.  **CommonAssetModule:** The network-aware component responsible for synchronizing asset changes with connected game clients. This handler invokes the module to push updates or removals, ensuring players receive the modified assets.

By encapsulating the logic for a specific asset category, this handler allows the core Asset Editor system to remain agnostic about the underlying storage and distribution mechanisms for different types of assets.

### Lifecycle & Ownership
-   **Creation:** An instance of CommonAssetTypeHandler is created during the server's bootstrap sequence. A higher-level manager, responsible for initializing the Asset Editor, instantiates handlers for each supported asset type (identified by file extension) and registers them in a central collection.
-   **Scope:** The object's lifetime is bound to the server session. It persists as long as the Asset Editor system is active and is reused for all operations on common assets.
-   **Destruction:** The instance is eligible for garbage collection upon server shutdown when the central handler registry is cleared. It does not manage any unmanaged resources and has no explicit destruction method.

## Internal State & Concurrency
-   **State:** This class is largely stateless, acting as a processor for incoming requests. Its primary function is to manipulate the external, static state held within `CommonAssetRegistry`. It contains a single piece of mutable internal state, `cachedDefaultUpdateQuery`, which is a minor performance optimization to avoid object reallocation.

-   **Thread Safety:** This class is **not thread-safe**. Its methods perform direct, unsynchronized mutations on the global `CommonAssetRegistry`. All invocations of this handler's methods must be serialized or executed on a dedicated, single-threaded service to prevent race conditions and data corruption within the asset system. The static nature of the target registry makes concurrent access extremely hazardous.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadAsset(path, dataPath, data, updateQuery, editorClient) | AssetLoadResult | O(log N) | Registers a new or updated common asset. The data is added to the CommonAssetRegistry. Complexity is dependent on the registry's internal implementation. |
| unloadAsset(path, updateQuery) | AssetLoadResult | O(log N) | Removes a common asset from the registry and triggers a network packet to inform clients of the removal. |
| restoreOriginalAsset(originalAssetPath, updateQuery) | AssetLoadResult | O(N + log N) | Reads the pristine asset from the source AssetPack on disk and uses it to overwrite the current version in the registry. Involves file I/O, where N is the size of the file. |
| getDefaultUpdateQuery() | AssetUpdateQuery | O(1) | Returns a default configuration for how clients should process this asset update, utilizing an internal cache. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic or external systems. It is an internal component of the Asset Editor. A central service dispatches requests to the appropriate registered handler based on the asset's type.

```java
// Hypothetical usage within a central AssetEditorService

// 1. An asset modification request arrives
AssetPath assetToModify = new AssetPath("myPack", "common/textures/stone.png");
byte[] newData = ...;

// 2. The service looks up the correct handler
AssetTypeHandler handler = handlerRegistry.getHandlerForPath(assetToModify); // Returns an instance of CommonAssetTypeHandler

// 3. The service delegates the operation to the handler
if (handler != null) {
    handler.loadAsset(assetToModify, tempPath, newData, query, client);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CommonAssetTypeHandler()`. Instances are managed by the Asset Editor's core initialization logic. Creating a rogue instance will result in a handler that is not registered and will never be used.
-   **Bypassing the Handler:** Do not call `CommonAssetRegistry.addCommonAsset()` directly. Doing so circumvents the complete logic defined in this handler, such as determining if an asset has actually changed. More critically, it would fail to trigger the `CommonAssetModule` to synchronize the changes with clients, leading to state desynchronization.

## Data Pipeline
The flow of data for a typical asset modification (e.g., saving a texture in the editor) follows this path:

> Flow:
> Asset Editor UI Action -> Network Packet -> Server's `EditorClient` -> **CommonAssetTypeHandler.loadAsset** -> `CommonAssetRegistry` Update -> Return `COMMON_ASSETS_CHANGED` -> `CommonAssetModule` -> Network Packet -> All Game Clients Update Asset Cache

