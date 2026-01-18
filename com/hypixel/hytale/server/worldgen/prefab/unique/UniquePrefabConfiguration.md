---
description: Architectural reference for UniquePrefabConfiguration
---

# UniquePrefabConfiguration

**Package:** com.hypixel.hytale.server.worldgen.prefab.unique
**Type:** Transient

## Definition
```java
// Signature
public class UniquePrefabConfiguration {
```

## Architecture & Concepts
The UniquePrefabConfiguration class is a foundational data object within the server-side world generation pipeline. It functions as an immutable "ruleset" or "blueprint" that defines all constraints and parameters for placing a single, specific *unique prefab*—such as a dungeon, monument, or special point of interest—into the game world.

Architecturally, this class decouples the placement *data* from the placement *logic*. The core world generation systems are agnostic of any specific prefab's rules; they simply operate on the contract provided by a UniquePrefabConfiguration instance. This allows designers to define and tune complex prefab placement behavior entirely through external asset files (e.g., JSON) without requiring any changes to the core engine code.

Each instance encapsulates a comprehensive set of conditions, including valid biomes, required terrain height, block placement masks, and spatial constraints relative to other world features. The world generator consumes these objects to determine if, where, and how a prefab should be instantiated during chunk generation.

## Lifecycle & Ownership
- **Creation:** Instances of UniquePrefabConfiguration are not created dynamically during gameplay. They are instantiated once during the server's bootstrap or asset loading phase. A central asset manager, responsible for parsing world generation configuration files, is the sole creator of these objects.

- **Scope:** An instance persists for the entire server session. It is effectively a singleton from the perspective of a specific prefab type, typically stored in a registry or map managed by the PrefabManager or a similar service.

- **Destruction:** The object is marked for garbage collection only when the server shuts down or a hot-reload of world generation assets is triggered, at which point the managing registry is cleared.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are declared as final and are assigned exclusively within the constructor. Once an instance is created, its state cannot be modified. This design is critical for ensuring predictable and deterministic world generation. The `exclusionRadiusSquared` field is a pre-calculated value, an optimization to avoid repeated square root or multiplication operations in performance-critical generation loops.

- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a UniquePrefabConfiguration object can be safely read by multiple world generation worker threads simultaneously without any risk of race conditions or data corruption. No locks or synchronization primitives are required.

    **Warning:** While the object itself is thread-safe, methods like `getRotation` accept a `Random` instance. The caller is responsible for ensuring that the provided `Random` object is used in a thread-safe manner if it is shared across threads.

## API Surface
The public API is composed almost entirely of non-blocking, constant-time accessors and evaluators.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isValidParentBiome(Biome) | boolean | O(1) | Evaluates if the prefab can spawn in the given Biome by checking against the internal biomeMask. |
| isValidParentBlock(int, int) | boolean | O(1) | Evaluates if the prefab can be placed on a specific block and fluid combination. |
| getPlacementConfiguration() | BlockMaskCondition | O(1) | Returns the condition object used to validate the immediate block footprint for placement. |
| getRotation(Random) | PrefabRotation | O(1) | Selects and returns a valid rotation for the prefab from its allowed list. Returns a default if none are configured. |
| getExclusionRadiusSquared() | double | O(1) | Returns the *squared* radius within which no other unique prefabs of this type may spawn. |

## Integration Patterns

### Standard Usage
The intended use is for a higher-level world generation service to retrieve a configuration from a registry and use it to validate potential placement locations.

```java
// A world generation service evaluating a potential spawn point
UniquePrefabConfiguration config = prefabRegistry.getConfiguration("ancient_ruin");
WorldGenContext context = new WorldGenContext(potentialX, potentialZ);

// Check high-level conditions first for fast rejection
if (config.isValidParentBiome(context.getBiome())) {
    // Perform more expensive checks
    if (config.getPlacementConfiguration().eval(context.getWorldView())) {
        // Conditions met, proceed with placement
        placePrefab(context, config);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Under no circumstances should this class be instantiated directly with `new UniquePrefabConfiguration()`. Doing so bypasses the asset loading system and hardcodes world generation rules into game logic, defeating the purpose of a data-driven architecture. Always retrieve configurations from the appropriate manager or registry.

- **State Assumption:** Do not cache the result of `getRotation`. This method is designed to be called once per placement attempt to provide variance. Caching its result will cause all instances of the prefab to have the same rotation.

## Data Pipeline
This class is a critical link in the data flow from configuration assets to the final world state.

> Flow:
> JSON Asset on Disk -> Server Asset Loader -> **UniquePrefabConfiguration** -> Unique Prefab Placement Service -> Voxel Buffer -> Final World Chunk

