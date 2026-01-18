---
description: Architectural reference for ICoordinateCondition
---

# ICoordinateCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ICoordinateCondition {
```

## Architecture & Concepts
The ICoordinateCondition interface defines a fundamental contract within Hytale's procedural generation library. It represents a stateless predicate function that evaluates to true or false based on a given set of world coordinates. This interface is a core building block for creating sophisticated and location-aware generation rules.

In essence, an implementation of ICoordinateCondition answers the question: "Does this specific location (x, y, z) meet a certain criterion?". This allows generators to compose complex behaviors, such as restricting biome placement to certain altitudes, confining ore veins to specific depth ranges, or preventing structures from spawning in predefined zones.

The presence of two `eval` methods, one with an additional integer parameter, suggests a mechanism for providing extra context, most likely a world seed. This enables deterministic, repeatable evaluations which are critical for consistent world generation. This interface is designed to be lightweight and frequently invoked, often millions of time during the generation of a single world region.

## Lifecycle & Ownership
As an interface, ICoordinateCondition itself does not have a lifecycle. The following pertains to concrete implementations of this contract.

- **Creation:** Implementations are typically instantiated by higher-level procedural generation systems, such as a rule parser or a biome definition loader. They are often created as part of a larger, composite rule structure and are not managed by a central service registry.
- **Scope:** The lifetime of an ICoordinateCondition object is tied to the procedural generation task that requires it. It may be short-lived, existing only for the duration of a single chunk generation, or it may be cached as part of a biome's definition for the server's lifetime.
- **Destruction:** Objects are eligible for garbage collection once the parent generator or rule set is discarded. There is no manual destruction or cleanup process.

## Internal State & Concurrency
- **State:** The contract strongly implies that implementations should be **immutable and stateless**. The `eval` methods are designed as pure functions where the output depends solely on the input coordinates. Storing mutable state within an implementation is a severe anti-pattern that can lead to non-deterministic and unpredictable world generation.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. The procedural generation engine is heavily parallelized, and a single ICoordinateCondition instance may be evaluated concurrently by multiple worker threads operating on different world chunks. Any implementation that is not thread-safe will introduce severe race conditions and world corruption.

## API Surface
The public contract consists of two evaluation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int x, int y, int z) | boolean | O(1) | Evaluates the condition for a 3D world coordinate. |
| eval(int x, int y, int z, int seed) | boolean | O(1) | Evaluates the condition for a 3D world coordinate using an additional context value, typically a world seed for deterministic noise. |

**Warning:** The complexity is assumed to be O(1) for simple mathematical checks. Implementations that perform more complex operations, such as sampling noise functions, will have a higher complexity. Performance is critical, as `eval` is a hot-path method.

## Integration Patterns

### Standard Usage
An ICoordinateCondition is typically used by a generator or placement system to guard an operation. The system iterates over a set of coordinates and uses the condition to decide whether to proceed.

```java
// A generator using a condition to determine placement
ICoordinateCondition heightCondition = new YCoordinateGreaterThan(64);
StructurePlacer placer = world.getStructurePlacer();

for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        int groundY = world.getTopSolidBlock(x, z);
        if (heightCondition.eval(x, groundY, z)) {
            placer.placeTree(x, groundY, z);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that modify their internal state within the `eval` method. This breaks determinism and thread safety.
- **World Lookups:** Do not perform expensive operations like reading block data from the world inside an `eval` call. The condition should be a fast, self-contained mathematical or logical check. Pass any required world data into the `eval` method if necessary, though this is not supported by the base interface.
- **Ignoring Thread Safety:** Never assume `eval` will be called from a single thread. All implementations must be re-entrant and free of side effects.

## Data Pipeline
This interface acts as a conditional gate within a larger data processing pipeline for world generation.

> Flow:
> Generation Algorithm -> Provides (X, Y, Z) Coordinates -> **ICoordinateCondition.eval()** -> Boolean Result -> [If True] -> Biome Painter / Structure Placer / Voxel Mutator

