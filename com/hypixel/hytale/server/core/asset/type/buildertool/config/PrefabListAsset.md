---
description: Architectural reference for PrefabListAsset
---

# PrefabListAsset

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config
**Type:** Data Asset Model

## Definition
```java
// Signature
public class PrefabListAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, PrefabListAsset>> {
```

## Architecture & Concepts
The PrefabListAsset class serves as a manifest for a collection of prefabs. It is not a prefab itself, but rather a data-driven asset that resolves high-level directory references into a concrete list of individual prefab file paths. This abstraction allows designers and developers to manage groups of related prefabs without explicitly listing every single file.

Its primary role is within the server-side asset system, providing a structured way for game systems like world generation or builder tools to query for available prefabs. The asset is defined in a JSON file and deserialized into this object representation at runtime.

The core of its functionality is defined by the static **CODEC** field. This `AssetBuilderCodec` dictates the JSON structure and maps it directly to the fields of the PrefabListAsset and its inner PrefabReference class. This codec is the definitive contract for the asset's file format.

The key architectural features are:
- **Indirect Referencing:** Instead of a flat list of files, the JSON asset defines `PrefabReference` objects. Each reference specifies a root directory, a sub-path, and a recursion flag.
- **Post-Load Processing:** The class uses an `afterDecode` hook in its codec. This triggers the `convertPrefabReferencesToPrefabPaths` method, which performs the crucial step of walking the file system to resolve the abstract references into a final, usable array of `Path` objects. This separates the declarative asset definition from the imperative file system discovery.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading framework via the defined **CODEC**. The `AssetStore` reads a corresponding JSON file from disk and uses the codec to instantiate and populate the object. Direct instantiation using `new` is an anti-pattern and will result in a non-functional object.
- **Scope:** A PrefabListAsset instance is cached within its corresponding `AssetStore` after being loaded. It persists for the entire server session, or until the asset cache is explicitly invalidated.
- **Destruction:** The object is eligible for garbage collection when the server shuts down or the `AssetStore` is cleared. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The object's state is mutable during the deserialization and post-processing phase. Fields like `prefabReferences` are intermediate representations. After the `afterDecode` hook completes, the final state, particularly the `prefabPaths` array, is intended to be treated as immutable by consumers. The object effectively transitions from a buildable state to a read-only state upon successful loading.

- **Thread Safety:** This class is **not** inherently thread-safe and requires careful handling in a concurrent environment.
    - **WARNING:** The static `getAssetStore` method contains a non-atomic check-then-act race condition (`if (ASSET_STORE == null)`). Calling this method concurrently from multiple threads during application startup will lead to unpredictable behavior and is strictly forbidden. It assumes initialization from a single, primary thread.
    - The `getRandomPrefab` method is safe for concurrent access as it uses `ThreadLocalRandom`.
    - All other getters are safe to call from multiple threads **only after** the asset has been fully loaded and initialized by the asset system. Modifying the array returned by `getPrefabPaths` is an anti-pattern that breaks this guarantee.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the singleton AssetStore for this asset type. **Not thread-safe on first call.** |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to retrieve the underlying asset map from the store. |
| getPrefabPaths() | Path[] | O(1) | Returns the final, resolved array of absolute paths to all contained prefabs. |
| getRandomPrefab() | Path | O(1) | Selects and returns a random prefab path from the resolved list. Thread-safe. |
| getId() | String | O(1) | Returns the unique identifier for this asset, derived from its filename. |

## Integration Patterns

### Standard Usage
A game system should retrieve a pre-loaded instance from the central `AssetStore` via its unique asset key. The system can then use the instance to get a list of all available prefabs or select one at random for spawning.

```java
// How a developer should normally use this
// Assumes AssetStore is already initialized on the main thread.
AssetStore<String, PrefabListAsset> store = PrefabListAsset.getAssetStore();
PrefabListAsset monsterPrefabs = store.get("hytale:monsters/overworld_tier1");

if (monsterPrefabs != null) {
    Path randomMonsterPrefabPath = monsterPrefabs.getRandomPrefab();
    if (randomMonsterPrefabPath != null) {
        // Pass the path to the PrefabStore to load and spawn the actual entity
        PrefabStore.get().spawnPrefab(world, position, randomMonsterPrefabPath);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabListAsset()`. The object will be empty and lack the critical resolved path data, which is only populated by the asset loading pipeline.
- **Concurrent Initialization:** Do not call `PrefabListAsset.getAssetStore()` from multiple threads simultaneously during server startup. This will cause a race condition. All asset store initialization should be serialized.
- **State Mutation:** Do not modify the array returned by `getPrefabPaths()`. This array is a reference to the internal cached state of the asset. Modifying it will affect all other systems that use this asset instance.

## Data Pipeline
The flow of data from a file on disk to a usable game object follows a clear, multi-stage process orchestrated by the asset system.

> Flow:
> JSON File (`.json`) -> `AssetStore` File Watcher -> JSON Deserializer -> **PrefabListAsset.CODEC** -> `PrefabReference` Instantiation -> File System Walk (Path Resolution) -> **PrefabListAsset** Hydration -> `AssetStore` Cache -> Game System Request -> `Path` returned to caller

---

