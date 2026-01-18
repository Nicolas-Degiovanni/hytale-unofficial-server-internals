---
description: Architectural reference for DistanceDensity
---

# DistanceDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class DistanceDensity extends Density {
```

## Architecture & Concepts
The DistanceDensity class is a specialized node within the procedural world generation framework. It operates as a component in a larger density function graph, which collectively defines the shape and substance of the game world before it is voxelized.

This class implements a simple but powerful concept: it maps a 3D point in space to a scalar density value based on its Euclidean distance from the world origin (0, 0, 0). The core architectural pattern employed here is the **Strategy Pattern**. The DistanceDensity node is not responsible for the specific mathematical transformation of the distance; instead, it delegates this logic to an injected *falloffFunction*. This decouples the geometric calculation (distance from origin) from the shaping logic (the falloff curve), making the component highly reusable for creating various spherical or radial density fields.

It serves as a foundational primitive for creating large-scale features like planets, floating islands, or protective bubbles where density changes relative to a central point.

## Lifecycle & Ownership
- **Creation:** An instance of DistanceDensity is created and configured by a higher-level system, typically a world generator or a density graph builder. The creator must provide a non-null Double2DoubleFunction during construction, which defines the node's behavior.
- **Scope:** The object's lifetime is bound to the density graph it is a part of. It is a transient object, created for a specific world generation task and is not persisted across sessions or shared globally.
- **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the density graph that references it is no longer in use.

## Internal State & Concurrency
- **State:** The internal state of a DistanceDensity instance is **immutable**. The single state-bearing field, falloffFunction, is private, final, and set only once via the constructor. The class does not cache results or modify its state during execution.

- **Thread Safety:** This class is **unconditionally thread-safe and re-entrant**. Its immutable nature ensures that the process method can be invoked by multiple world-generation worker threads concurrently without any risk of race conditions or data corruption.

    **WARNING:** While the DistanceDensity class itself is thread-safe, the provided falloffFunction must also be thread-safe. Supplying a stateful, non-thread-safe function will violate the concurrency guarantees of this component.

## API Surface
The public contract is inherited from the parent Density class and consists of a single processing method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(1) | Calculates the density at the given context position. Throws NullPointerException if the context is null. |

## Integration Patterns

### Standard Usage
DistanceDensity is designed to be instantiated with a specific falloff behavior, often defined as a lambda. It is then used as a node within a larger composite density function.

```java
// Example: Create a simple spherical density field that is 1.0 at the center
// and linearly falls off to 0.0 at a distance of 512 units.
double maxRadius = 512.0;
Double2DoubleFunction linearFalloff = distance -> {
    if (distance >= maxRadius) {
        return 0.0;
    }
    return 1.0 - (distance / maxRadius);
};

// The node is configured once and can be reused for countless evaluations.
Density sphere = new DistanceDensity(linearFalloff);

// A world generator would then call process() for many different positions.
double densityAtPoint = sphere.process(new Density.Context(100.0, 50.0, 0.0));
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas:** Avoid providing a falloffFunction that relies on or modifies external state. This breaks the immutability and thread-safety of the component, leading to unpredictable and non-deterministic world generation.

    ```java
    // ANTI-PATTERN: The falloff function modifies external state.
    // This will produce incorrect results in a multithreaded environment.
    final double[] counter = {0.0};
    Double2DoubleFunction badFunction = distance -> {
        counter[0] += 1.0; // Race condition!
        return 1.0 - distance;
    };
    Density unsafeNode = new DistanceDensity(badFunction);
    ```

- **Null Injection:** The constructor contract requires a non-null function. Passing null will not fail at construction but will cause a NullPointerException during the first call to the process method.

## Data Pipeline
DistanceDensity acts as a transformation node in a data flow. It consumes positional data and produces a scalar density value.

> Flow:
> World Generator -> Density.Context (Position) -> **DistanceDensity.process()** -> double (Raw Density) -> Subsequent Graph Nodes

