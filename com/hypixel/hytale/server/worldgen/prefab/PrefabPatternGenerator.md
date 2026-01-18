---
description: Architectural reference for PrefabPatternGenerator
---

# PrefabPatternGenerator

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPatternGenerator {
```

## Architecture & Concepts

The PrefabPatternGenerator is a passive, immutable data object that serves as a blueprint for procedural prefab placement within the world generator. It does not perform any generation itself; instead, it encapsulates a complete set of rules, conditions, and strategies that a higher-level orchestrator uses to determine where and how to place a specific category of prefabs.

Architecturally, this class embodies the **Strategy Pattern** through composition. It aggregates numerous functional interfaces (IPointGenerator, ICoordinateCondition, etc.) into a single, cohesive configuration unit. Each instance represents a distinct "recipe" for placing structures, such as dungeons in a specific biome or trees on a certain terrain type.

This design decouples the high-level generation loop from the specific rules for each prefab type, allowing for a data-driven and highly extensible world generation system. New prefab patterns can be defined in external configuration files and loaded at runtime without altering the core generation engine.

## Lifecycle & Ownership

-   **Creation:** Instances are not intended to be created directly in code. They are instantiated by a deserialization process during server startup, which parses world generation configuration files (e.g., JSON). A central registry or factory is responsible for loading and holding these objects.
-   **Scope:** An instance persists for the entire server session. Once loaded from configuration, it is treated as a read-only definition, referenced by the world generation system whenever new chunks are created.
-   **Destruction:** The object is garbage collected when the server shuts down or when the world generation configuration is reloaded. No explicit cleanup is required.

## Internal State & Concurrency

-   **State:** **Immutable**. All fields are declared final and are populated exclusively through the constructor. The state of a PrefabPatternGenerator cannot be altered after its creation.
-   **Thread Safety:** **Inherently Thread-Safe**. Due to its immutability, a single instance can be safely read by multiple world generation threads concurrently without any risk of race conditions or data corruption. This is a critical feature for achieving high performance in a multi-threaded chunk generation environment.

## API Surface

The public API consists entirely of accessors for its internal strategies and configuration values. It provides the contract that the world generator uses to execute the placement logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCategory() | PrefabCategory | O(1) | Returns the category of prefabs this pattern applies to. |
| getGridGenerator() | IPointGenerator | O(1) | Provides the core strategy for generating candidate placement points. |
| getMapCondition() | ICoordinateCondition | O(1) | Provides a broad-phase filter, often used for biome or region checks. |
| getHeightCondition() | ICoordinateRndCondition | O(1) | Provides a fine-grained filter for validating placement at a specific altitude. |
| getPrefabPlacementConfiguration() | BlockMaskCondition | O(1) | Defines the required block patterns at the target location for placement to succeed. |
| getDisplacement(seed, x, z) | int | O(1) | Calculates a deterministic, random positional offset for the prefab. |

## Integration Patterns

### Standard Usage

The PrefabPatternGenerator is intended to be retrieved from a central registry and used by a world generation service. The service iterates through its strategies to validate a potential placement location.

```java
// A world generator service would retrieve and use the pattern
PrefabPatternGenerator pattern = worldGenRegistry.getPatternFor("dungeon_crypt");
IPointGenerator pointGenerator = pattern.getGridGenerator();

// Generate candidate points for a given chunk
List<Point3D> candidates = pointGenerator.getPointsForChunk(chunkPos);

for (Point3D candidate : candidates) {
    // Execute the pattern's conditions to validate the point
    if (pattern.getMapCondition().check(candidate) && pattern.getHeightCondition().check(candidate)) {
        // If valid, proceed with placement
        placePrefab(candidate, pattern);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new PrefabPatternGenerator()`. This defeats the purpose of a data-driven system and hard-codes world generation logic. All patterns should be defined in external configuration files.
-   **State Modification:** Do not attempt to modify the object's fields via reflection. The immutability of this class is a core design principle for ensuring thread safety and predictable generation. Violating this contract will lead to severe and difficult-to-diagnose concurrency bugs.

## Data Pipeline

This class does not process data itself; it provides the logic for a stage in the larger world generation pipeline. Its components are invoked in a sequence to filter potential placement locations.

> Flow:
> Chunk Coordinates -> **getGridGenerator()** -> Candidate Points -> **getMapCondition()** -> Biome-Filtered Points -> **getHeightCondition()** -> Vertically-Validated Points -> **getPrefabPlacementConfiguration()** -> Final Validated Location -> Prefab Placer

