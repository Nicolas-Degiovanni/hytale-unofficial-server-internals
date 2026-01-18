---
description: Architectural reference for ScriptedBrushAsset
---

# ScriptedBrushAsset

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes
**Type:** Data Asset

## Definition
```java
// Signature
public class ScriptedBrushAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ScriptedBrushAsset>> {
```

## Architecture & Concepts
The ScriptedBrushAsset is a data-oriented class that represents a configured, reusable builder tool brush. It is not an active component; rather, it serves as a data template or "prefab" that defines a sequence of actions, or BrushOperations, to be performed.

This class is a managed asset, meaning its lifecycle and instantiation are controlled by the Hytale AssetRegistry. Each instance corresponds to a specific JSON asset file on disk. The static CODEC field defines the contract for deserializing the JSON data into a hydrated ScriptedBrushAsset object in memory. This process is handled automatically by the engine's asset loading system.

The primary role of a ScriptedBrushAsset is to act as a configuration provider for a BrushConfigCommandExecutor. The executor is the active entity that performs the work; the asset simply provides the list of instructions. This decouples the definition of a brush (the asset) from its execution (the executor), allowing for complex brushes to be defined entirely in data files without requiring new Java code.

A key feature is its ability to compose brushes. A ScriptedBrushAsset can contain a LoadOperationsFromAssetOperation, which recursively loads and integrates the operations from another ScriptedBrushAsset. This allows for the creation of modular and reusable brush components.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the AssetRegistry during the engine's asset loading phase. The public constructor exists only to satisfy the requirements of the AssetBuilderCodec. Direct instantiation via the `new` keyword will result in a non-functional object.
- **Scope:** An instance of ScriptedBrushAsset persists for the entire duration that its parent asset store is loaded. This is typically the entire game session. All loaded assets are cached in a static DefaultAssetMap for fast, repeated access.
- **Destruction:** Instances are garbage collected when the AssetRegistry is cleared or reloaded, for example, on game shutdown or when resource packs are changed.

## Internal State & Concurrency
- **State:** The internal state, particularly the `operations` list, is populated once during asset deserialization. While the object is technically mutable, it must be treated as **effectively immutable** after loading. Modifying its state at runtime can lead to unpredictable behavior across the entire application, as the same instance is shared globally.
- **Thread Safety:** This class is **not thread-safe**. The static `getAssetMap` method contains a lazy initializer that is not protected against race conditions. It is assumed that asset loading and initialization are completed in a single-threaded context before any other systems attempt to access assets. Instance methods are also not safe for concurrent access, as they operate on a standard, non-synchronized List. All interactions with ScriptedBrushAsset instances should occur on the main game thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(String id) | static ScriptedBrushAsset | O(1) | Retrieves a cached asset instance by its unique ID. Returns null if not found. |
| getId() | String | O(1) | Returns the unique identifier of this asset. |
| getOperations() | List<BrushOperation> | O(1) | Returns a direct reference to the internal list of operations. **Warning:** Do not modify this list. |
| loadIntoExecutor(executor) | void | O(N) | Populates the provided executor with the operations defined in this asset. N is the number of operations. This is the primary integration point. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a pre-loaded asset from the static map using its ID and then pass it to an executor to apply its configuration.

```java
// Retrieve the executor for the current context
BrushConfigCommandExecutor executor = getActiveExecutor();

// Look up the desired brush asset by its string identifier
ScriptedBrushAsset brushAsset = ScriptedBrushAsset.get("hytale:stone_pillar_brush");

// Apply the asset's configuration to the executor
if (brushAsset != null) {
    brushAsset.loadIntoExecutor(executor);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ScriptedBrushAsset()`. The resulting object will be uninitialized, lack an ID, and contain no operations. It will be useless and may cause NullPointerExceptions.

- **Runtime State Modification:** Do not modify the list returned by `getOperations()`. This list is a shared global resource. Modifying it will affect every single part of the game that uses this brush asset, leading to severe and hard-to-debug side effects.

    ```java
    // DO NOT DO THIS
    ScriptedBrushAsset asset = ScriptedBrushAsset.get("some_brush");
    if (asset != null) {
        // This modifies a globally cached asset and will break other systems
        asset.getOperations().clear();
    }
    ```

## Data Pipeline
The ScriptedBrushAsset acts as a bridge between static data defined in files and the live, executable systems in the game engine.

> Flow:
> JSON Asset File -> AssetRegistry Loader -> AssetBuilderCodec -> **ScriptedBrushAsset** (In-Memory Cache) -> `loadIntoExecutor()` -> BrushConfigCommandExecutor -> Game World State Change

