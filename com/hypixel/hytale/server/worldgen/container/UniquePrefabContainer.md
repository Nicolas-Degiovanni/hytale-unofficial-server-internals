---
description: Architectural reference for UniquePrefabContainer
---

# UniquePrefabContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient

## Definition
```java
// Signature
public class UniquePrefabContainer {
```

## Architecture & Concepts
The UniquePrefabContainer is a high-level orchestrator within the server-side world generation pipeline. Its primary function is to manage and execute a collection of UniquePrefabGenerator instances to produce a deterministic set of placement instructions for special, non-repeating world features like dungeons, landmarks, or quest locations.

This class acts as a configuration holder for a specific group of prefabs that are intended to be generated together within a world zone. It encapsulates the logic for seeding the random number generator and iterating through each configured generator, ensuring that they can operate with an awareness of each other's potential placements. This is critical for enforcing rules like exclusion zones, preventing unique landmarks from spawning on top of one another.

The final output, an array of UniquePrefabEntry objects, is not the world data itself, but rather a complete and immutable *placement plan*. This plan contains all necessary information—position, rotation, prefab asset reference, and placement conditions—for a later stage of world generation (the population or "stamping" phase) to apply these structures to the world grid.

## Lifecycle & Ownership
- **Creation:** A UniquePrefabContainer is instantiated by a higher-level world generation service, such as a ZoneGenerator, during the initialization phase for a world region. Its configuration, specifically the array of UniquePrefabGenerators, is typically loaded from world generation asset files.
- **Scope:** This object is short-lived and designed for a single, specific generation task. It is created, its generate method is invoked once to produce a placement plan, and it is then typically discarded. It does not persist in memory after the generation pass is complete.
- **Destruction:** The object holds no native resources and is managed entirely by the Java Garbage Collector. No explicit cleanup is necessary.

## Internal State & Concurrency
- **State:** Immutable. The internal fields, seedOffset and the generators array, are marked as final and are set exclusively at construction time. The object's state cannot be modified after it is created.
- **Thread Safety:** This class is inherently thread-safe. The generate method is a pure function relative to the object's state; it creates a new Random instance for each invocation, preventing contention. Multiple threads can safely call the generate method on a shared instance, provided the ChunkGenerator argument is itself thread-safe.

**WARNING:** While the class is immutable, the UniquePrefabGenerator array passed to the constructor is not defensively copied. Modifying this array externally after the container has been created will lead to undefined behavior and is a violation of the class contract.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, position, chunkGenerator) | UniquePrefabEntry[] | O(N * M) | Orchestrates the generation of all contained prefabs. N is the number of generators; M is the complexity of each generator's placement-finding algorithm. |
| getGenerators() | UniquePrefabGenerator[] | O(1) | Returns a direct reference to the internal array of generators. |

## Integration Patterns

### Standard Usage
The container is used by a world generation orchestrator to calculate placements for a zone. The resulting plan is then passed to a system responsible for modifying chunk data.

```java
// In a world generation service...
UniquePrefabContainer landmarkContainer = loadContainerFromConfig("zone_A_landmarks");
ChunkGenerator chunkGen = getChunkGeneratorForArea(area);

// Generate the placement plan for all landmarks in this container
UniquePrefabContainer.UniquePrefabEntry[] placementPlan = landmarkContainer.generate(worldSeed, area.getCenter(), chunkGen);

// Pass the plan to the world populator to apply the changes
worldPopulator.applyPrefabEntries(placementPlan);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually construct a UniquePrefabContainer. These should be defined in data files and loaded by the world generation framework to ensure consistency.
- **State Modification:** Do not retain a reference to the generator array passed into the constructor and modify it later. This breaks the immutability contract and can cause non-deterministic generation.
- **Redundant Generation:** Do not call the generate method multiple times for the same seed and area. The process is deterministic and computationally expensive; the result should be cached if needed.

## Data Pipeline
The UniquePrefabContainer transforms a world seed and terrain context into a concrete placement plan.

> Flow:
> World Seed & Zone Config -> **UniquePrefabContainer** -> (iterates over) UniquePrefabGenerator -> (queries) ChunkGenerator -> UniquePrefabEntry[] (Placement Plan) -> World Populator -> Final Chunk Data

