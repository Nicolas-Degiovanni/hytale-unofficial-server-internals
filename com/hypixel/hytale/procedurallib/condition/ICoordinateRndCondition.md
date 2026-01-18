---
description: Architectural reference for ICoordinateRndCondition
---

# ICoordinateRndCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface ICoordinateRndCondition {
   boolean eval(int var1, int var2, int var3, int var4, Random var5);
}
```

## Architecture & Concepts
ICoordinateRndCondition is a functional interface that defines a core contract within the procedural generation framework. It represents a stateless predicate function used to evaluate a specific condition at a given 4D coordinate, incorporating a pseudo-random element.

This interface is a manifestation of the **Strategy Pattern**. It decouples high-level procedural generation algorithms (e.g., biome placement, ore vein distribution, structure spawning) from the specific, low-level rules that govern them. By passing concrete implementations of this interface to generators, the system can dynamically alter world generation logic without modifying the core generation orchestrators.

The design strongly encourages pure, deterministic functions. All required context—coordinates and a seeded Random instance—is passed directly into the `eval` method. This ensures that for a given world seed, the outcome of any condition check is repeatable, which is fundamental for consistent world generation.

## Lifecycle & Ownership
As an interface, ICoordinateRndCondition does not have a lifecycle itself. The lifecycle pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated by configuration loaders or factory methods at the start of a world generation pass. They are often created as lightweight, transient objects.
- **Scope:** The lifetime of an implementation instance is generally scoped to a single, specific generation task. It is used, potentially across many coordinate evaluations, and then becomes eligible for garbage collection once the task is complete.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup requirements.

## Internal State & Concurrency
- **State:** The interface contract is inherently stateless. Implementations **must not** maintain mutable internal state. Storing state within an implementation would violate the principle of referential transparency and could lead to non-deterministic or incorrect world generation, especially in a multi-threaded environment.
- **Thread Safety:** The interface is thread-safe by design. Because all context is passed via method arguments, a single instance of an implementation can be safely shared and executed by multiple worker threads simultaneously during parallelized world generation.

**WARNING:** Developers implementing this interface must ensure their `eval` method is re-entrant and free of side effects. Avoid accessing global singletons or any shared mutable state.

## API Surface
The public contract consists of a single method, defining the evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int, int, int, int, Random) | boolean | O(impl) | Evaluates a procedural condition at a 4D coordinate. Returns true if the condition is met, false otherwise. The complexity is entirely dependent on the specific implementation. |

## Integration Patterns

### Standard Usage
Implementations of this interface are intended to be passed to procedural generation services or algorithms. The generator invokes the `eval` method to make decisions for each coordinate it processes.

```java
// A generator receives a condition implementation to guide its logic.
public class StructurePlacer {
    public void placeStructures(World world, ICoordinateRndCondition placementCondition) {
        Random seededRandom = new Random(world.getSeed());
        for (int x = 0; x < 1024; x++) {
            for (int z = 0; z < 1024; z++) {
                // The generator uses the condition to make a decision.
                if (placementCondition.eval(x, 64, z, 0, seededRandom)) {
                    // Place a structure here...
                }
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating an implementation that stores results or modifies its own fields during an `eval` call. This breaks thread safety and determinism.
- **Ignoring the Random Argument:** Failing to use the provided Random instance for any stochastic calculations. This will cause all parallel generation threads to produce identical random sequences, leading to patterned and unnatural results. Always use the `var5` parameter for randomness.
- **External Dependencies:** The `eval` method should not call out to external systems, read from files, or access global state. It must be a self-contained, pure function.

## Data Pipeline
This component acts as a decision point or filter within a larger data generation pipeline.

> Flow:
> Generation Orchestrator -> Provides (Coordinates, World Seed) -> **ICoordinateRndCondition.eval()** -> Returns Boolean Decision -> Generation Orchestrator -> Places Block / Spawns Entity / Skips

