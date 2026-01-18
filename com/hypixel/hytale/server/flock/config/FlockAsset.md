---
description: Architectural reference for FlockAsset
---

# FlockAsset

**Package:** com.hypixel.hytale.server.flock.config
**Type:** Asset Definition

## Definition
```java
// Signature
public abstract class FlockAsset implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, FlockAsset>> {
```

## Architecture & Concepts
The FlockAsset class is an abstract base for defining the behavioral properties of an NPC flock. It is not a live game entity but rather an immutable data blueprint loaded from configuration files at server startup. This class and its concrete implementations form a critical bridge between game design data (JSON files) and the server's NPC spawning logic.

The core architectural pattern is **data-driven polymorphism**. The static field CODEC, an AssetCodecMapCodec, acts as a factory. It inspects a type identifier within the source JSON and delegates deserialization to the appropriate concrete subclass, such as RangeSizeFlockAsset or WeightedSizeFlockAsset. This allows designers to create new flock behaviors by simply adding a new JSON file and registering a corresponding subclass, without modifying the core spawning systems.

All FlockAsset instances are managed by a global, server-wide AssetStore, which is a specialized collection retrieved from the central AssetRegistry. The static method getAssetStore serves as the sole entry point for accessing these configurations, ensuring that all game systems reference a single, consistent set of flock definitions.

## Lifecycle & Ownership
-   **Creation:** FlockAsset instances are created exclusively by the Hytale Asset System during the server's initial loading phase. The AssetCodecMapCodec deserializes flock definition JSON files into concrete FlockAsset objects. Direct instantiation by game code is strictly forbidden.
-   **Scope:** An instance of a FlockAsset is a global singleton for its given ID. It persists for the entire lifetime of the server process. It is loaded once and never reloaded.
-   **Destruction:** All FlockAsset instances are destroyed and garbage collected when the server shuts down and the AssetRegistry is cleared. There is no mechanism for hot-reloading or individual destruction.

## Internal State & Concurrency
-   **State:** **Effectively Immutable**. Once a FlockAsset is deserialized and loaded into the AssetStore, its state is considered final. Its fields (maxGrowSize, blockedRoles) are intended for read-only access by game logic.

    **WARNING:** The blockedRoles field is a raw array. Modifying its contents after retrieval is a severe anti-pattern that will introduce unpredictable behavior across the entire server, as the object reference is shared globally.

-   **Thread Safety:** **Conditionally Thread-Safe**.
    -   **Reads:** The object is safe for concurrent reads from any thread after the server's asset loading phase is complete.
    -   **Initialization:** The static getAssetStore method uses lazy initialization. Accessing this method from multiple threads *before* the AssetStore is initialized can lead to a race condition. However, in standard operation, the asset system is initialized synchronously on the main server thread at startup, mitigating this risk.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, server-wide store for all FlockAsset definitions. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Convenience method to directly access the underlying map of assets from the store. |
| getId() | String | O(1) | Returns the unique identifier for this asset, corresponding to its key in the JSON map. |
| getMinFlockSize() | abstract int | O(N) | Returns the minimum number of NPCs in a newly spawned flock. Complexity is implementation-dependent. |
| pickFlockSize() | abstract int | O(N) | Selects a random flock size based on the rules of the concrete implementation. |
| getMaxGrowSize() | int | O(1) | Returns the maximum size a flock can grow to after its initial spawn. |
| getBlockedRoles() | String[] | O(1) | Returns an array of NPC roles forbidden from joining this flock. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a specific FlockAsset definition from the global AssetStore via its unique ID. This object is then used to provide parameters to the NPC spawning system.

```java
// Retrieve the central store of flock definitions
IndexedLookupTableAssetMap<String, FlockAsset> flockAssets = FlockAsset.getAssetMap();

// Get a specific flock definition by its ID (e.g., "hytale:wolf_pack")
FlockAsset wolfPackDefinition = flockAssets.get("hytale:wolf_pack");

if (wolfPackDefinition != null) {
    // Use the definition to determine spawn count
    int spawnCount = wolfPackDefinition.pickFlockSize();
    npcSpawner.spawnFlock(location, "hytale:wolf", spawnCount);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new RangeSizeFlockAsset()`. The asset system is the sole owner of the object's lifecycle. Direct instantiation bypasses the asset registry, leading to unmanaged and inconsistent state.
-   **State Mutation:** Do not modify the array returned by getBlockedRoles. This shared state will affect all subsequent spawns using this definition, leading to difficult-to-diagnose bugs.
-   **Premature Access:** Do not call getAssetStore or getAssetMap before the server's main asset loading sequence is complete. This will result in a null reference or an empty map.

## Data Pipeline
The flow of data from configuration file to in-game use is a one-way process managed by the asset system.

> Flow:
> JSON File on Disk (`*.json`) -> AssetStore Loader -> AssetCodecMapCodec (Deserialization) -> **FlockAsset Instance** -> IndexedLookupTableAssetMap (In-Memory Cache) -> NPC Spawning System (Read-Only Access)

