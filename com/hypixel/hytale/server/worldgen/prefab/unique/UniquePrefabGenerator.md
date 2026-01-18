---
description: Architectural reference for UniquePrefabGenerator
---

# UniquePrefabGenerator

**Package:** com.hypixel.hytale.server.worldgen.prefab.unique
**Type:** Transient

## Definition
```java
// Signature
public class UniquePrefabGenerator {
```

## Architecture & Concepts

The UniquePrefabGenerator is a specialized, rule-based placement engine within the server-side procedural world generation pipeline. Its sole responsibility is to determine a valid 3D coordinate for placing a "unique" prefab, such as a major dungeon, a landmark structure, or a critical point of interest.

Each instance of this class is configured for a single category of unique prefab and operates as a self-contained placement strategy. It embodies a "scatter and test" algorithm: it generates a candidate position within a configured area and then rigorously validates that position against a series of environmental and spatial constraints. These constraints include biome type, terrain height, distance from zone borders, proximity to other unique prefabs, and underlying noise map values.

A critical design feature is its built-in fallback system. If the generator fails to find a suitable location after a set number of attempts, it will invoke a `forceGeneration` routine. This ensures that essential world features are always placed, preventing a catastrophic world generation failure at the cost of potentially non-ideal placement. This resilience is paramount for stable and predictable world creation.

## Lifecycle & Ownership

-   **Creation:** Instances are created by a higher-level world generation orchestrator, such as a `ZoneGenerator`. The configuration for each generator is typically loaded from world generation asset files, which define the rules for a specific zone or world.
-   **Scope:** The lifetime of a UniquePrefabGenerator object is ephemeral and bound to a specific placement task. It is created, its `generate` method is invoked once, and it is then immediately eligible for garbage collection. It holds no state that persists beyond this single operation.
-   **Destruction:** The object is managed by the Java Garbage Collector. It is destroyed once the calling world generation process releases its reference, which typically occurs after the returned position has been consumed.

## Internal State & Concurrency

-   **State:** The object's state is **effectively immutable**. All core configuration fields are declared `final` and are injected via the constructor. The generator does not modify its own state during its operation. Its methods are pure functions with respect to its initial configuration.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. Its core placement algorithms depend on a `java.util.Random` instance, which is not thread-safe. All interactions with a UniquePrefabGenerator instance must be confined to the single thread responsible for the current world generation task to ensure deterministic and predictable outcomes.

## API Surface

The public API is minimal, exposing only the primary generation entry point and accessors for its configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(...) | Vector3i | O(N) | The primary entry point. Attempts to find a valid placement position. N is the `maxFailed` parameter. Throws no exceptions but logs failures. |
| generatePrefab(random) | WorldGenPrefabSupplier | O(1) | Selects a prefab variant from the weighted map. Returns null if the map is empty. |
| getConfiguration() | UniquePrefabConfiguration | O(1) | Returns the immutable configuration object for this generator. |

## Integration Patterns

### Standard Usage

The generator is intended to be used by a world generation service. The service retrieves a configured instance, invokes the `generate` method to get a position, and then uses that position to schedule the actual placement of the chosen prefab.

```java
// Assume 'worldGenContext' provides access to the current generation state
UniquePrefabGenerator dungeonGenerator = worldGenContext.getUniqueGenerator("ancient_dungeon");
UniquePrefabContainer.UniquePrefabEntry[] existingPrefabs = worldGenContext.getPlacedUniquePrefabs();
Random random = worldGenContext.getRandom();
int seed = worldGenContext.getSeed();

// Find a valid position for the dungeon
Vector3i placementPosition = dungeonGenerator.generate(
    seed,
    null, // Let the generator find a random position
    worldGenContext.getChunkGenerator(),
    random,
    100, // Max attempts
    existingPrefabs
);

// Schedule the prefab for placement at the returned coordinate
WorldGenPrefabSupplier prefabToPlace = dungeonGenerator.generatePrefab(random);
worldGenContext.getPrefabPlacer().schedule(prefabToPlace, placementPosition);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new UniquePrefabGenerator()`. Instances should be constructed by a factory or builder that loads the correct `UniquePrefabConfiguration` from game assets. Manual construction risks creating invalid or misconfigured generators.
-   **State Reuse:** Do not reuse a generator instance across different world seeds or for different generation tasks. Its behavior is tied to the `Random` and `ChunkGenerator` instances passed into the `generate` method, and it is not designed for reuse.
-   **Concurrent Access:** **WARNING:** Never call the `generate` method from multiple threads on the same instance. This will lead to race conditions within the `Random` object and produce non-deterministic, corrupt world generation results.

## Data Pipeline

The `generate` method executes a validation pipeline to determine if a candidate location is suitable. Data flows from broad spatial checks to fine-grained block-level analysis.

> Flow:
> 1.  **Candidate Selection:** A random coordinate (x, z) is generated within a circle defined by `configuration.anchor` and `configuration.maxDistance`.
> 2.  **Exclusion Zone Check:** The coordinate is checked against the exclusion radii of all previously placed unique prefabs. **(FAIL FAST)**
> 3.  **Noise Map Check:** The coordinate is evaluated against a `MapCondition` to ensure it is in a valid region of the underlying noise map (e.g., a continental plate). **(FAIL FAST)**
> 4.  **Zone & Biome Query:** The `ChunkGenerator` is queried to retrieve the Zone and Biome at the coordinate.
> 5.  **Zone & Biome Validation:** The system validates that the coordinate is in the correct Zone, is not too close to a zone border, and is in a valid parent biome. **(FAIL FAST)**
> 6.  **Height Calculation:** The terrain surface height (y) is calculated, respecting water levels if configured.
> 7.  **Height Validation:** The full (x, y, z) coordinate is evaluated against a `HeightCondition`. **(FAIL FAST)**
> 8.  **Parent Block Validation:** The block type at and just below the surface is checked against a list of valid parent blocks. **(FAIL FAST)**
> 9.  **Success:** If all checks pass, the final `Vector3i` is returned.
> 10. **Retry/Force:** If any check fails, the pipeline is re-run from step 1. If `maxFailed` is exceeded, the `forceGeneration` fallback is triggered.

