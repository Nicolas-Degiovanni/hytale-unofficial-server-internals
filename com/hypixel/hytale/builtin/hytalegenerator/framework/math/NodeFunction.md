---
description: Architectural reference for NodeFunction
---

# NodeFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Stateful Utility

## Definition
```java
// Signature
public class NodeFunction implements Function<Double, Double>, Double2DoubleFunction {
```

## Architecture & Concepts
The NodeFunction class is a foundational mathematical primitive within the Hytale world generation framework. It represents a one-dimensional, piecewise linear function defined by a series of discrete control points, or nodes. Its primary role is to provide a configurable mapping from an input value to an output value through linear interpolation.

This component is frequently used to define "shaping functions" or "transfer curves" that modulate the output of other systems. For example, it can be used to:
*   Define the elevation profile of terrain based on a noise value.
*   Control the density of vegetation or ore distribution at different altitudes.
*   Create smooth transitions between different biome parameters.

Internally, the class maintains a sorted list of (input, output) coordinate pairs. When queried with an input value, it performs an efficient search to locate the two nearest points and calculates the corresponding output value along the line segment connecting them. Values outside the defined range of points are clamped to the output value of the nearest endpoint.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its default constructor, for example `new NodeFunction()`. It is then configured by chaining calls to the `addPoint` method. This class follows a builder-like pattern for its initial setup.
- **Scope:** The object's lifetime is typically transient and bound to the scope of the specific generator or algorithm that creates it. It is not a globally shared service and does not persist beyond the generation task it was created for.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as all references to it are released.

## Internal State & Concurrency
- **State:** The NodeFunction is a stateful and highly mutable object during its configuration phase. It maintains two internal lists:
    1.  **points:** A list of `double[]` arrays representing the (x, y) coordinates. This list is re-sorted after every call to `addPoint`.
    2.  **ranges:** A cached list of `RangeDouble` objects, derived from the `points` list. This serves as a search acceleration structure, allowing for logarithmic-time lookups. This cache is rebuilt every time a point is added.

- **Thread Safety:** This class is **not thread-safe**.
    - **Write Operations:** Concurrent calls to `addPoint` will lead to race conditions, potentially causing a `ConcurrentModificationException` or leaving the internal state corrupted and unsorted. All configuration of a NodeFunction instance must be performed from a single thread or be protected by external synchronization.
    - **Read/Write Operations:** A call to `get` while another thread is calling `addPoint` can produce unpredictable results or throw an exception, as the underlying lists may be modified during the search operation.
    - **Recommendation:** Treat instances of NodeFunction as immutable after their initial configuration is complete. Fully populate the function with all required points before passing it to multi-threaded processing systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addPoint(double in, double out) | NodeFunction | O(N log N) | Adds a new control point. Triggers a full sort of all existing points, making it an expensive operation. |
| get(double input) | double | O(log N) | Calculates the output value for a given input via linear interpolation. Uses an efficient binary search on cached ranges. |
| apply(Double input) | Double | O(log N) | Implementation of the standard Java Function interface. Delegates directly to the `get` method. |
| contains(double x) | boolean | O(N) | Checks for the existence of a point with the given x-coordinate. Scans the entire point list. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate, configure, and then use the object for repeated calculations. The configuration phase should be treated as a distinct step before the object is used in performance-critical code.

```java
// 1. Instantiate the function
NodeFunction heightCurve = new NodeFunction();

// 2. Configure the function by adding control points
// This defines a curve that is flat at 0, rises to 100, then drops to 80.
heightCurve.addPoint(0.0, 0.0);
heightCurve.addPoint(0.5, 100.0);
heightCurve.addPoint(1.0, 80.0);

// 3. Use the configured function in a processing loop
// The object should now be treated as read-only.
for (double noiseValue : noiseSamples) {
    double terrainHeight = heightCurve.get(noiseValue);
    // ... use terrainHeight
}
```

### Anti-Patterns (Do NOT do this)
- **Modification After Sharing:** Do not modify a NodeFunction instance after it has been passed to another system, especially a multi-threaded one. The lack of thread safety makes this extremely hazardous.
- **Per-Call Instantiation:** Do not create a new NodeFunction inside a tight loop. The object is designed to be configured once and reused many times.
- **High-Frequency Point Addition:** Avoid calling `addPoint` repeatedly in a performance-sensitive loop. Each call incurs an `O(N log N)` sorting penalty. If points must be generated programmatically, generate them into a temporary collection first and then add them to the NodeFunction.

## Data Pipeline
NodeFunction acts as a transformation step within a larger data processing pipeline, typically in procedural generation. It does not manage a pipeline itself.

> Flow:
> Generator Configuration -> **NodeFunction.addPoint()** -> (Immutable Function State) -> **NodeFunction.get(input)** -> Transformed Output Value

