---
description: Architectural reference for CheckTagWorldHeightRadiusProvider
---

# CheckTagWorldHeightRadiusProvider

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.worldlocationproviders
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class CheckTagWorldHeightRadiusProvider extends WorldLocationProvider {
```

## Architecture & Concepts
The CheckTagWorldHeightRadiusProvider is a concrete implementation of the **WorldLocationProvider** strategy. Its architectural role is to act as a data-driven spatial query engine within the Adventure Mode objective system. It is designed to find a suitable in-world coordinate by performing a targeted surface search for blocks that possess specific, developer-defined tags.

This component is not a long-lived service but rather a lightweight, configurable object that encapsulates a single, specific location-finding rule. Its behavior is defined entirely by data loaded from asset configuration files, typically as part of a larger quest or objective definition.

The core of its operation is a spiral search algorithm that efficiently scans a circular area. For each coordinate pair (X, Z) in the spiral, it queries the world for the highest solid block and checks its asset tags against a pre-compiled list. This "top-down" approach makes it ideal for finding locations on the world's surface, such as a suitable place to spawn an entity or a point of interest.

A key architectural feature is its integration with the Hytale Codec and AssetRegistry systems. The static **CODEC** field manages deserialization from configuration files. During this process, it transforms human-readable string tags (e.g., "spawnable_surface") into highly optimized integer-based tag indexes. This translation occurs once at load time, ensuring that the runtime execution of the search is highly performant and avoids repeated string comparisons.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via their constructor in game logic. They are instantiated by the Hytale Codec system during the deserialization of adventure objective assets. The static **CODEC** field dictates the entire creation and validation process.
-   **Scope:** The object's lifetime is bound to the parent objective or configuration that defines it. It persists in memory as long as that configuration is active. Once created and its internal tag indexes are resolved, its state is effectively immutable.
-   **Destruction:** The object is eligible for garbage collection when its parent configuration is unloaded from memory. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is stateful, containing the search radius and the list of block tags. However, it is **effectively immutable** after initialization. The `runCondition` method is a pure function with respect to the object's state; it reads from its configuration fields but never modifies them.
-   **Thread Safety:** The class itself contains no locks or mutable shared state, making it inherently safe to be referenced from multiple threads. The `runCondition` method performs read-only operations on the provided World object. Therefore, the overall thread safety of an operation involving this class is contingent upon the thread safety guarantees of the World and WorldChunk APIs it is given. It is safe to assume this provider can be used in parallelized server tasks, provided the world access is managed correctly.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| runCondition(World world, Vector3i position) | Vector3i | O(N), where N is radius^2 | Executes the spatial query. Scans a circular area for a surface block matching the configured tags. Returns the coordinate *above* the found block (Y+1) or null if no match is found. |

## Integration Patterns

### Standard Usage
This provider is not intended to be invoked directly. It is executed by higher-level systems, such as an objective or procedural spawning system, which holds a polymorphic reference to a WorldLocationProvider.

```java
// A system retrieves a provider, which was deserialized from an asset file.
WorldLocationProvider locationRule = objectiveConfig.getSpawnLocationProvider();

// The system executes the rule to find a concrete location.
// The origin position is often a player or a zone anchor.
Vector3i spawnPoint = locationRule.runCondition(server.getWorld(), originPosition);

if (spawnPoint != null) {
    // A valid location was found. Proceed to spawn an entity or create a structure.
    world.spawnEntity(EntityType.GOLEM, spawnPoint);
} else {
    // Handle the failure case where no suitable location could be found.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CheckTagWorldHeightRadiusProvider()`. This bypasses the critical `CODEC` logic that validates configuration and, more importantly, resolves string-based tags into optimized integer indexes via the AssetRegistry. An improperly constructed instance will fail at runtime.
-   **State Mutation:** Do not use reflection or other means to modify the `radius` or `blockTags` fields after the object has been deserialized. The internal state, particularly the `blockTagsIndexes` array, will become inconsistent, leading to undefined behavior.
-   **Large Radii:** Using an excessively large radius can cause significant performance degradation, as the search complexity is quadratic. This can lead to server-side lag spikes when the condition is executed. **Warning:** Profile any usage with a radius greater than 100 chunks.

## Data Pipeline
The flow of data from configuration to a final world position is a multi-stage process orchestrated by several core engine systems.

> Flow:
> Adventure Objective Asset File (JSON/HOCON) -> Hytale Codec Deserializer -> **CheckTagWorldHeightRadiusProvider Instance** (with resolved tag indexes) -> Objective Execution Engine -> `runCondition` call with World reference -> World/Chunk Data Access -> Result (Vector3i or null) -> Consuming System (e.g., Spawner)

