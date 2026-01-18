---
description: Architectural reference for OffsetPointGenerator
---

# OffsetPointGenerator

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Transient

## Definition
```java
// Signature
public class OffsetPointGenerator implements IPointGenerator {
```

## Architecture & Concepts
The OffsetPointGenerator is a concrete implementation of the **Decorator** design pattern. Its sole purpose is to wrap an existing IPointGenerator instance and transparently apply a fixed spatial offset to all coordinate-based queries. It does not generate points itself; it delegates this responsibility to the wrapped generator.

This class functions as a coordinate space translator. It allows a single underlying point generation algorithm, such as a Poisson disk sampler, to be logically reused in different locations within the world without modifying the core algorithm. This is a fundamental component for procedural generation systems, enabling the composition of complex worlds by shifting and placing pre-defined features. For example, a generator for village well locations can be defined once and then wrapped by an OffsetPointGenerator to place that well in any village, regardless of the village's world coordinates.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor: `new OffsetPointGenerator(...)`. It is not managed by a service locator or dependency injection framework. It is typically created by higher-level procedural logic, such as a BiomeGenerator or FeaturePlacer, that requires a translated set of points.
- **Scope:** Short-lived. The lifetime of an OffsetPointGenerator is tightly coupled to the specific generation task that created it. It is a lightweight, single-purpose object.
- **Destruction:** The object is eligible for garbage collection as soon as all references to it are dropped, which typically occurs after a single procedural generation pass completes. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the reference to the decorated generator and the double-precision offset values, are marked as final. They are assigned only once during construction. The object's state cannot be modified after creation.
- **Thread Safety:** **Conditionally Thread-Safe**. The thread safety of an OffsetPointGenerator is entirely dependent on the thread safety of the IPointGenerator instance it decorates. This class introduces no mutable state or synchronization primitives of its own. If the wrapped generator is thread-safe, any instance of OffsetPointGenerator wrapping it will also be safe for concurrent use.

## API Surface
The API contract is identical to the IPointGenerator interface it implements. All methods delegate to the wrapped generator after transforming input coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(seed, x, y) | ResultBuffer.ResultBuffer2d | O(decorated) | Finds the nearest point(s) by forwarding the query to the wrapped generator with translated coordinates (x + offsetX, y + offsetY). |
| nearest3D(seed, x, y, z) | ResultBuffer.ResultBuffer3d | O(decorated) | Finds the nearest point(s) by forwarding the query to the wrapped generator with translated coordinates (x + offsetX, y + offsetY, z + offsetZ). |
| collect(...) | void | O(decorated) | Collects all points within a bounding box. The query box is passed unmodified, but the points returned by the wrapped generator are translated before being passed to the final consumer. |
| getInterval() | double | O(1) | Retrieves the characteristic interval directly from the wrapped generator. The offset has no effect on this value. |

## Integration Patterns

### Standard Usage
The canonical use case is to wrap an existing generator to shift its output into a new coordinate space.

```java
// 1. Define a base point generator, for example a Poisson disk sampler.
IPointGenerator baseGenerator = new PoissonPointGenerator(seed, 16.0);

// 2. Create a shifted version of the generator for a specific world region.
//    This instance effectively moves the entire point set by (1024, 0, 2048).
IPointGenerator shiftedGenerator = new OffsetPointGenerator(baseGenerator, 1024.0, 0.0, 2048.0);

// 3. All queries to the shiftedGenerator are now in the new coordinate space.
//    A query for a point at (5, 10, 15) in the shifted space...
ResultBuffer.ResultBuffer3d result = shiftedGenerator.nearest3D(seed, 5, 10, 15);

//    ...is transparently converted to a query at (1029, 10, 2063) on the base generator.
```

### Anti-Patterns (Do NOT do this)
- **Redundant Chaining:** Avoid nesting multiple OffsetPointGenerators. While functionally correct, it introduces unnecessary object overhead and call stack depth.

  ```java
  // BAD: Inefficient and hard to read.
  IPointGenerator gen1 = new OffsetPointGenerator(base, 100, 0, 0);
  IPointGenerator gen2 = new OffsetPointGenerator(gen1, 50, 0, 0);

  // GOOD: Pre-calculate the final offset and use a single decorator.
  IPointGenerator finalGen = new OffsetPointGenerator(base, 150, 0, 0);
  ```
- **Wrapping Null:** The constructor does not perform null checks. Passing a null generator will result in a NullPointerException at the first method invocation, not at construction time. Always ensure the decorated generator is a valid instance.

## Data Pipeline
The OffsetPointGenerator acts as a pure transformation step within a larger data flow. It modifies coordinates but does not alter the underlying point distribution logic.

**For `nearest` queries:**
> Flow:
> Input Coordinates -> **OffsetPointGenerator** (Adds offset to coordinates) -> Decorated IPointGenerator -> ResultBuffer

**For `collect` queries:**
> Flow:
> Input Bounding Box -> Decorated IPointGenerator -> Raw Point Data -> **OffsetPointGenerator** (Lambda adds offset to each point) -> Final PointConsumer

