---
description: Architectural reference for CellValueReturnType
---

# CellValueReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class CellValueReturnType extends ReturnType {
```

## Architecture & Concepts
The CellValueReturnType class is a specialized strategy node within the procedural world generation's density function graph. Its primary architectural role is to perform a **conditional, position-shifted density sampling**.

Instead of evaluating a density function at the primary sample position provided by the context, this class uses a pre-calculated proxy position, `closestPoint0`. It then delegates the final density calculation to a separate, injected density function, the `sampleField`, using this new position.

This pattern is fundamental for creating features based on proximity to a calculated geometric structure, such as a Voronoi cell boundary or a procedural spline. For example, it allows the generator to determine terrain material not by the absolute coordinates of a block, but by its proximity to the center of a noise cell, and then sampling a different noise function at that cell's center.

If the proxy position `closestPoint0` is not available (is null), the class acts as a critical fallback mechanism, returning a stable `defaultValue`. This design choice ensures the integrity of the density graph, preventing null pointer exceptions or undefined behavior from propagating through the generation pipeline.

### Lifecycle & Ownership
-   **Creation:** Instantiated during the construction of the main density graph, typically by a parent density node that calculates cellular or positional data (e.g., a Voronoi or Worley noise node). It is configured once with its dependencies and is considered part of the static generator definition.
-   **Scope:** The object's lifetime is bound to the world generator's density graph. It persists in memory as long as the generator configuration is active.
-   **Destruction:** Marked for garbage collection when the world generator instance is destroyed, for example, upon server shutdown or when loading a different world.

## Internal State & Concurrency
-   **State:** This class is **immutable**. Its internal fields, `sampleField` and `defaultValue`, are final and are injected via the constructor. The `get` method does not modify any internal state, operating only on its arguments.
-   **Thread Safety:** The class is **conditionally thread-safe**. The `get` method itself is re-entrant and safe for concurrent invocation. Its overall thread safety depends on the guarantees provided by the injected `sampleField` object. The method promotes safe concurrency by creating a new, temporary `Density.Context` for each call, preventing state pollution between threads that might share the parent context.

## API Surface
The public contract is exclusively defined by the `get` method inherited from its parent `ReturnType`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePosition, closestPoint0, closestPoint1, context) | double | O(sampleField) | Calculates a density value. If `closestPoint0` is null, returns the configured default. Otherwise, it samples the injected `sampleField` at the `closestPoint0` position. The complexity is entirely dependent on the injected `Density` function. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay logic. It is a low-level component configured within a larger density graph, typically defined in a world generation asset file. A parent node, such as one for cellular noise, would construct it.

```java
// Hypothetical configuration within a parent density node
// This code would exist inside another generator class, not called directly.

// 1. Define the density function to be sampled at the cell's center
Density oreVeinDensity = new PerlinNoise(...);
double fallbackValue = 0.0; // Default density if no cell is found

// 2. Create the CellValueReturnType strategy
// This object is now configured to sample oreVeinDensity at a proxy location.
ReturnType returnType = new CellValueReturnType(oreVeinDensity, fallbackValue, threadCount);

// 3. The parent node would later invoke it during processing
// double finalDensity = returnType.get(..., closestPoint, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Do not manually create and call an instance of this class. It is designed to be driven by a parent density node that provides the correct positional arguments, especially `closestPoint0`.
-   **Null Dependency:** Passing a null `sampleField` to the constructor violates its non-null contract and will lead to a `NullPointerException` at runtime. The system relies on this dependency being valid.
-   **Stateful Dependencies:** Injecting a `Density` object into `sampleField` that is not thread-safe can break the concurrency model and lead to severe, difficult-to-diagnose generation artifacts.

## Data Pipeline
The class acts as a routing and delegation node within a larger data flow. It transforms an input context and positional data into a final density value.

> Flow:
> Parent Density Node -> Provides `closestPoint0` and `Context` -> **CellValueReturnType** -> (If `closestPoint0` is valid) -> Creates new `Context` with `position = closestPoint0` -> Injected `sampleField.process()` -> Returns `double`
>
> **OR**
>
> Parent Density Node -> Provides `null` `closestPoint0` -> **CellValueReturnType** -> Returns `defaultValue` as `double`

