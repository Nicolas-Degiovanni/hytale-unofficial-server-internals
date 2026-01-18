---
description: Architectural reference for Pattern
---

# Pattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class Pattern {
```

## Architecture & Concepts
The Pattern class is an abstract, foundational component of the procedural world generation system. It represents a single, evaluatable rule or condition within a larger generator configuration. Its primary purpose is to answer the question: "Does a specific condition hold true at this point in the world?"

This class embodies the Strategy design pattern. Concrete implementations define specific algorithms for matching features, such as terrain formations, resource deposits, or locations suitable for placing structures. The world generator iterates over a set of these Pattern objects, invoking their `matches` method for coordinates within a VoxelSpace to make decisions about block placement and world composition.

The `Pattern.Context` inner class serves as a Data Transfer Object (DTO), bundling all necessary spatial and environmental information required for a pattern to perform its evaluation. This design decouples the pattern's logic from the generator's main loop, promoting modular and reusable generation rules. The inclusion of a `WorkerIndexer.Id` in the context is a critical design choice, indicating that the entire pattern evaluation system is designed for high-throughput, parallel execution across multiple worker threads.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of Pattern are instantiated by higher-level generator systems, typically during the loading phase of a world or biome profile. The static factory methods `noPattern` and `yesPattern` provide singleton-like instances for trivial true/false conditions.
- **Scope:** A Pattern object's lifetime is tied to the generator configuration that references it. As they are intended to be stateless, they are typically created once and reused for millions of evaluations throughout a world generation task.
- **Destruction:** Instances are eligible for garbage collection when the world generator or biome profile they belong to is unloaded.

## Internal State & Concurrency
- **State:** The abstract Pattern class is stateless. Concrete implementations are **strongly expected** to be immutable or effectively immutable after their initial configuration. All data required for evaluation is passed via the `Pattern.Context` parameter in the `matches` method.
- **Thread Safety:** **CRITICAL:** Implementations of this class **must be thread-safe**. The generator engine will invoke the `matches` method concurrently from multiple worker threads, as indicated by the `WorkerIndexer.Id` field in the context. Storing mutable state within a Pattern instance without proper synchronization will lead to severe race conditions and non-deterministic world generation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Context) | boolean | Varies | The core evaluation method. Returns true if the pattern's conditions are met at the context's position. Complexity is implementation-dependent. |
| readSpace() | SpaceSize | O(1) | Returns the bounding box of voxel data this pattern needs to read to perform its evaluation. This is a performance contract with the generator. |
| noPattern() | Pattern | O(1) | Static factory for a pattern that always returns false. |
| yesPattern() | Pattern | O(1) | Static factory for a pattern that always returns true. |

## Integration Patterns

### Standard Usage
A developer extends the Pattern class to define a new world generation rule. The implementation is then registered with a generator system, which will invoke it during its processing loop.

```java
// 1. Define a concrete pattern for finding flat ground
public class FlatGroundPattern extends Pattern {
    private final SpaceSize requiredSpace = new SpaceSize(new Vector3i(-2, 0, -2), new Vector3i(2, 0, 2));

    @Override
    public boolean matches(@Nonnull Pattern.Context context) {
        // Logic to check for flat ground within the requiredSpace
        // using context.materialSpace and context.position.
        return isAreaFlat(context);
    }

    @Override
    public SpaceSize readSpace() {
        return this.requiredSpace;
    }
}

// 2. Register the pattern with the generator (conceptual)
worldGenerator.addPlacementRule(new FlatGroundPattern(), structureToPlace);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not add mutable fields to a Pattern subclass to track state between calls to `matches`. This will break when used in a multithreaded context and produce unpredictable results.
- **Incorrect Space Declaration:** The `readSpace` method must return a `SpaceSize` that fully encloses all voxel data accessed by the `matches` method. Declaring a smaller space than what is actually read can cause the generator to provide an incomplete `VoxelSpace`, leading to data access violations or exceptions.
- **Ignoring Context:** Do not attempt to access global state or other services from within a `matches` implementation. All necessary information is provided in the `Pattern.Context`. Violating this principle breaks the hermetic, parallel-safe nature of the system.

## Data Pipeline
The data flow for a Pattern evaluation is driven entirely by the world generator.

> Flow:
> World Generator Loop -> Creates **Pattern.Context** for (X,Y,Z) -> Invokes **Pattern.matches(context)** -> Receives boolean result -> Generator makes placement decision

