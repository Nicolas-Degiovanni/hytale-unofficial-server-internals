---
description: Architectural reference for AssetTypeHandler
---

# AssetTypeHandler

**Package:** com.hypixel.hytale.builtin.asseteditor.assettypehandler
**Type:** Abstract Strategy

## Definition
```java
// Signature
public abstract class AssetTypeHandler {
```

## Architecture & Concepts
The AssetTypeHandler is an abstract base class that forms the cornerstone of the Asset Editor's extensibility. It implements the **Strategy Pattern**, defining a common interface for handling the lifecycle of different categories of game assets (e.g., textures, models, sounds).

Each concrete subclass of AssetTypeHandler encapsulates the specific logic required to load, unload, and manage a particular asset type. This design decouples the high-level Asset Editor orchestration logic from the low-level, engine-specific implementation details of asset manipulation. For example, a TextureHandler would contain logic for GPU texture uploads, while a SoundHandler would interface with the audio engine.

The core responsibility of a handler is to translate generic asset modification requests into concrete actions within the game engine, often by constructing and dispatching an AssetUpdateQuery.

### Lifecycle & Ownership
- **Creation:** Concrete implementations are instantiated once per asset type during the initialization of the Asset Editor subsystem. A central registry or factory is responsible for creating and mapping each handler to its corresponding AssetEditorAssetType configuration.
- **Scope:** An AssetTypeHandler instance is a long-lived object. Its lifetime is bound to the Asset Editor session itself, not to any individual asset. It persists from editor startup to shutdown.
- **Destruction:** Handlers are de-referenced and eligible for garbage collection when the Asset Editor system is shut down. They do not have an explicit destruction or cleanup method in their public contract.

## Internal State & Concurrency
- **State:** The base class is primarily configured with immutable state, such as the AssetEditorAssetType config and the rootPath, which are set at construction time. However, it contains one piece of mutable state: the **cachedDefaultUpdateQuery** field. This suggests that generating the default query may be a non-trivial operation, and the result is cached to improve performance on subsequent calls.

- **Thread Safety:** **This class is not thread-safe.** The presence of the mutable `cachedDefaultUpdateQuery` field without any synchronization primitives (e.g., locks, volatile) creates a potential for race conditions.

    **WARNING:** All interactions with an AssetTypeHandler instance and its subclasses must be confined to a single, designated thread, typically the main engine or editor thread. Accessing a handler from multiple threads will lead to unpredictable behavior and data corruption.

## API Surface
The public contract is defined by abstract methods that must be implemented by subclasses, along with convenience wrappers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadAsset(path, dataPath, data, query, client) | AssetLoadResult | O(N) | **Core Method.** Loads asset data into the engine. Complexity depends on asset size and engine work. |
| unloadAsset(path, query) | AssetLoadResult | O(N) | **Core Method.** Removes or deactivates an asset from the engine. |
| restoreOriginalAsset(path, query) | AssetLoadResult | O(N) | **Core Method.** Reverts an asset to its original state from the asset store. |
| getDefaultUpdateQuery() | AssetUpdateQuery | O(1) Amortized | Returns a default, often cached, query object for asset updates. |
| isValidData(data) | boolean | O(1) | A hook for subclasses to perform quick validation on raw asset data. |

## Integration Patterns

### Standard Usage
A handler should never be invoked directly. The central Asset Editor service is responsible for routing requests to the appropriate handler based on the asset's type.

```java
// Pseudo-code for an AssetEditor service
// 1. Get the asset path and determine its type
AssetPath assetPath = new AssetPath("models/monsters/creeper.json");
AssetEditorAssetType assetType = assetTypeRegistry.getTypeForPath(assetPath);

// 2. Retrieve the correct handler from a central registry
AssetTypeHandler handler = handlerRegistry.getHandler(assetType);

// 3. Delegate the load operation to the handler
byte[] assetData = readFile(assetPath);
handler.loadAsset(assetPath, dataPath, assetData, editorClient);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ConcreteAssetHandler()`. Handlers are managed by a central system that injects their configuration. Bypassing this system will result in uninitialized and non-functional handlers.
- **Storing Per-Asset State:** A handler is designed to be a stateless processor for an entire *category* of assets. Storing state related to a *specific* asset (e.g., `lastLoadedCreeperModel`) inside the handler is a critical design flaw that breaks reusability and introduces memory leaks.
- **Cross-Thread Invocation:** As detailed under Concurrency, invoking handler methods from worker threads is strictly forbidden and will cause state corruption.

## Data Pipeline
The AssetTypeHandler acts as a critical processing stage, translating high-level editor commands into low-level engine updates.

> Flow:
> Editor UI Event -> EditorClient -> Asset Management Service -> **AssetTypeHandler** -> AssetUpdateQuery -> AssetStore -> Engine Subsystems (Renderer, Audio)

