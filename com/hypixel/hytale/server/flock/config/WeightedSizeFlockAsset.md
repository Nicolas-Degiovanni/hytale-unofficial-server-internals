---
description: Architectural reference for WeightedSizeFlockAsset
---

# WeightedSizeFlockAsset

**Package:** com.hypixel.hytale.server.flock.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class WeightedSizeFlockAsset extends FlockAsset {
```

## Architecture & Concepts
The WeightedSizeFlockAsset is a data-driven configuration model that defines a specific strategy for determining the size of an entity flock at spawn time. It is a concrete implementation of the abstract FlockAsset, specializing in a weighted probability distribution for size selection.

This class acts as a bridge between server configuration files (e.g., JSON assets) and the server-side entity spawning system. Its primary role is to encapsulate the logic for calculating a flock's initial population based on a set of predefined weights, allowing designers to finely tune the rarity of different group sizes.

The static CODEC field is the most critical architectural feature. It leverages the Hytale Codec system to declaratively map raw data from an asset file into a strongly-typed Java object. This pattern decouples the spawning logic from the specifics of data storage and validation, ensuring that any loaded WeightedSizeFlockAsset instance is guaranteed to be in a valid state.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are instantiated and hydrated by the Hytale asset loading pipeline, which uses the static BuilderCodec field to deserialize the asset file. The framework calls the protected no-args constructor and subsequently populates the fields based on the asset data.
- **Scope:** An instance of this class persists in memory for as long as its corresponding asset is required by the server. This is typically for the entire duration of a server session or the lifetime of a specific world zone where this flock type can spawn.
- **Destruction:** The object is managed by the Java garbage collector. It is de-referenced and cleaned up when the Asset Management system unloads the relevant assets, such as during a server shutdown or a world change.

## Internal State & Concurrency
- **State:** The internal state consists of *minSize* and *sizeWeights*. This state is **effectively immutable** after the object is deserialized. The fields are populated once during creation by the codec system and are not modified thereafter.
- **Thread Safety:** This class is **thread-safe for read operations**. Multiple threads can safely call its public methods simultaneously. The core state does not change, and the `pickFlockSize` method relies on utility functions that are expected to be safe for concurrent use.

**WARNING:** The `getSizeWeights` method returns a direct reference to the internal array. Modifying this array externally will break the immutability contract and lead to unpredictable behavior. It should be treated as a read-only structure.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMinSize() | int | O(1) | Returns the configured absolute minimum size for the flock. |
| getSizeWeights() | double[] | O(1) | Returns the array of weights used for size calculation. |
| getMinFlockSize() | int | O(1) | Implements the base contract to return the minimum possible flock size. |
| pickFlockSize() | int | O(N) | Calculates and returns a random flock size based on the configured weights. N is the length of the sizeWeights array. |

## Integration Patterns

### Standard Usage
This asset is intended to be retrieved from a central asset or configuration manager and used by a higher-level spawning service to determine entity counts.

```java
// SpawningSystem retrieves the asset configuration
FlockAsset flockConfig = assetManager.get("my_mob_flock.json");

// The system can now use the config to determine spawn count
if (flockConfig instanceof WeightedSizeFlockAsset) {
    int countToSpawn = flockConfig.pickFlockSize();
    spawnEntities(mobType, countToSpawn);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WeightedSizeFlockAsset()`. This creates an empty, uninitialized object with a null `sizeWeights` array, which will cause a NullPointerException when `pickFlockSize` is called. Always allow the asset system to create and manage these objects.
- **State Mutation:** Do not modify the array returned by `getSizeWeights`. This violates the object's immutability and will affect all subsequent calls to `pickFlockSize` across the entire server, leading to difficult-to-diagnose bugs.

## Data Pipeline
The primary flow for this class is during the server's asset loading phase, not during real-time gameplay data processing.

> Flow:
> Asset File on Disk (JSON) -> Server Asset Loader -> **BuilderCodec Deserializer** -> **WeightedSizeFlockAsset Instance** -> In-Memory Asset Registry -> Entity Spawning Service (on-demand access)

