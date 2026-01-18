---
description: Architectural reference for ReachLocationMarkerAsset
---

# ReachLocationMarkerAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.reachlocation
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class ReachLocationMarkerAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ReachLocationMarkerAsset>> {
```

## Architecture & Concepts

The ReachLocationMarkerAsset class is a data-driven **Asset Type**, not a service or manager. It serves as the in-memory representation of a "reach location" objective marker, which is defined externally in a JSON file. Its primary architectural function is to provide a strict schema and a runtime container for data used by the adventure mode and questing systems.

The core of this class is the static **CODEC** field. This `AssetBuilderCodec` instance acts as a blueprint for deserialization and validation. It dictates that any corresponding JSON asset must contain a `Radius` (float > 0) and a `Name` (non-empty string). This declarative approach ensures data integrity at the asset loading boundary, preventing malformed data from entering the game engine.

Instances of ReachLocationMarkerAsset are not managed individually. Instead, they are loaded and indexed by the central **AssetRegistry** into a type-specific **AssetStore**. Game systems do not interact with the asset files directly; they query the `AssetStore` using a string-based key to retrieve a fully-formed, validated asset object. This decouples game logic from the specifics of data storage and loading.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale Asset Loading system during engine initialization or asset hot-reloading. The static `CODEC` is invoked to parse a corresponding JSON file on disk and construct a new ReachLocationMarkerAsset object. Manual instantiation is strictly forbidden.
-   **Scope:** An instance of this asset persists for the entire game session. Once loaded into the `AssetStore`, it is treated as globally accessible, read-only data. Its lifecycle is bound to the `AssetStore` itself, not to any specific game object or scene.
-   **Destruction:** All instances are de-referenced and eligible for garbage collection when the `AssetStore` for this type is cleared. This typically occurs only on game shutdown or during a full asset reload initiated by developers.

## Internal State & Concurrency

-   **State:** An instance of ReachLocationMarkerAsset is effectively **immutable** post-initialization. Its fields (`id`, `name`, `radius`) are populated once by the `CODEC` during deserialization and are not designed to be changed at runtime. It functions as a read-only data transfer object.
-   **Thread Safety:** The object is **conditionally thread-safe**. As a read-only data container, its getter methods can be safely called from any thread after the asset loading process is complete.

    **Warning:** The static `getAssetStore` method uses a non-synchronized, lazy initialization pattern. While this is generally safe in Hytale's architecture where asset stores are initialized on the main thread during startup, calling this method from multiple threads simultaneously before the store is initialized could lead to a race condition.

## API Surface

The public API is minimal, focusing on data retrieval from the global store or the instance itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton repository for all ReachLocationMarkerAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to get the underlying map of assets from the store. |
| getId() | String | O(1) | Returns the unique asset identifier, derived from its file path or key. |
| getName() | String | O(1) | Returns the human-readable name of the marker, as defined in the asset file. |
| getRadius() | float | O(1) | Returns the trigger radius for the location marker. |

## Integration Patterns

### Standard Usage

Game systems should always retrieve assets from the central `AssetStore`. Never cache the store itself in long-lived objects; request it when needed.

```java
// Retrieve the central repository for all ReachLocationMarker assets
AssetStore<String, ReachLocationMarkerAsset, ?> store = ReachLocationMarkerAsset.getAssetStore();

// Fetch a specific marker by its unique ID (e.g., "my_quest_tower_entrance")
ReachLocationMarkerAsset marker = store.get("my_quest_tower_entrance");

if (marker != null) {
    // Use the asset's data to drive game logic
    float objectiveRadius = marker.getRadius();
    String objectiveName = marker.getName();
    // ...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ReachLocationMarkerAsset()`. The resulting object will be uninitialized, invalid, and not registered in the `AssetStore`, making it invisible to the engine and likely to cause NullPointerExceptions.
-   **Runtime Modification:** Do not attempt to modify the state of a retrieved asset instance (e.g., via reflection). Assets are shared, global data. Modifying one instance will affect all systems using it, leading to unpredictable and difficult-to-debug behavior.
-   **Assuming Existence:** Do not assume an asset exists. Always check for null after retrieving an asset from the store, as the requested key may be invalid or the asset may have failed to load.

## Data Pipeline

The flow of data from disk to game logic is governed by the engine's asset system. This class sits at the end of the deserialization process, acting as the structured, in-memory representation of the source data.

> Flow:
> JSON File on Disk (`.json`) -> Engine Asset Loader -> **ReachLocationMarkerAsset.CODEC** (Deserialization & Validation) -> **ReachLocationMarkerAsset Instance** -> Populated into `AssetStore` -> Queried by Game Logic (e.g., Objective System)

