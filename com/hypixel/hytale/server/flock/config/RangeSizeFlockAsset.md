---
description: Architectural reference for RangeSizeFlockAsset
---

# RangeSizeFlockAsset

**Package:** com.hypixel.hytale.server.flock.config
**Type:** Transient Data Object

## Definition
```java
// Signature
public class RangeSizeFlockAsset extends FlockAsset {
```

## Architecture & Concepts
The RangeSizeFlockAsset is a server-side data-driven configuration object that defines the properties of an entity "flock" whose population is determined by a random value. It extends the base FlockAsset, specializing it to handle scenarios where a group of entities, upon spawning, should have a variable size selected from a specified minimum and maximum range.

This class is a key component of the asset system and is not intended for direct manipulation by game logic. Its primary role is to be deserialized from configuration files (e.g., JSON) via its static CODEC field. The Hytale Codec system uses this definition to validate and instantiate flock configurations during server startup or world loading. This architecture allows designers and developers to define complex spawning behaviors in data files without modifying engine code.

It represents one of several possible flock sizing strategies, contrasting with fixed-size or other procedurally determined flock configurations.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec framework during the asset loading process. The static CODEC field is the entry point for deserialization from a data source into a live Java object. The static factory method getUnknownFor is a fallback mechanism used by the asset system to create a placeholder object when a definition is missing or invalid.
- **Scope:** An instance of RangeSizeFlockAsset persists for the lifetime of the server's asset registry. It is loaded once and cached, shared across all systems that may need to spawn this particular type of flock.
- **Destruction:** The object is eligible for garbage collection only when the server's asset registry is cleared, typically during a server shutdown or a full resource reload. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The core state is the mutable `size` integer array, which holds the minimum and maximum bounds for the flock size. While technically mutable, it is designed to be written to once during deserialization and treated as immutable thereafter. A static DEFAULT_SIZE provides a safe fallback.
- **Thread Safety:** This class is **not thread-safe**. The internal `size` array is not protected against concurrent modification. However, the intended operational pattern is "write-once, read-many". Assets are loaded and configured in a single-threaded context at startup. Afterward, methods like pickFlockSize may be called from multiple threads (e.g., different world simulation threads). This is safe only if the post-initialization immutability contract is honored.

**WARNING:** Modifying the state of a loaded RangeSizeFlockAsset at runtime will introduce severe race conditions and non-deterministic behavior, as the same instance is shared globally.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSize() | int[] | O(1) | Returns a direct reference to the internal size array. |
| getMinFlockSize() | int | O(1) | Returns the minimum possible size for a spawned flock. |
| pickFlockSize() | int | O(1) | Calculates and returns a random flock size within the configured range. |
| getUnknownFor(String id) | RangeSizeFlockAsset | O(1) | Static factory to create a default, non-functional asset for a given ID. |

## Integration Patterns

### Standard Usage
This asset is not retrieved from a service registry. It is accessed via an AssetManager or a similar system using its unique identifier. A higher-level spawning system uses it to determine entity counts.

```java
// A SpawningSystem retrieves the asset by its ID (e.g., "hytale:flock_pigeon")
FlockAsset asset = AssetManager.get(FlockAsset.class, "hytale:flock_pigeon");

// The system then uses the asset's contract to determine spawn count
if (asset instanceof RangeSizeFlockAsset) {
    int countToSpawn = asset.pickFlockSize();
    // ... logic to spawn 'countToSpawn' entities
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RangeSizeFlockAsset()`. The object is unconfigured and will not function correctly. All creation must be handled by the asset loading pipeline via the CODEC.
- **Post-Load Mutation:** Never modify the array returned by `getSize()`. Doing so affects all subsequent spawns using this asset across the entire server and is not thread-safe.

```java
// DO NOT DO THIS
RangeSizeFlockAsset asset = (RangeSizeFlockAsset) AssetManager.get(FlockAsset.class, "hytale:flock_pigeon");

// This will cause unpredictable behavior globally and is a race condition
asset.getSize()[1] = 9999; 
```

## Data Pipeline
The RangeSizeFlockAsset is the *result* of a data pipeline, not an active participant in one. It represents loaded and parsed configuration data.

> Flow:
> Flock Definition File (JSON) -> Server Asset Loader -> **RangeSizeFlockAsset.CODEC** -> In-Memory RangeSizeFlockAsset Instance -> Entity Spawning System

