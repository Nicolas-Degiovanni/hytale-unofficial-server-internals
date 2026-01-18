---
description: Architectural reference for CircleIterator
---

# CircleIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Transient

## Definition
```java
// Signature
public class CircleIterator implements Iterator<Vector3d> {
```

## Architecture & Concepts
The CircleIterator is a specialized, high-performance generator for creating a sequence of equidistant points on the circumference of a horizontal circle (a circle on the XZ plane). It is a fundamental building block within the Hytale math library, designed to support procedural generation, entity placement, AI patrol pathing, and visual effect patterns.

By implementing the standard Java Iterator interface, it provides a familiar and efficient contract for consuming these points. It encapsulates the underlying trigonometric calculations, abstracting away the complexity of converting angles and radii into world-space coordinates.

This class is optimized for performance-critical contexts. It likely relies on the engine's TrigMathUtil, which often uses pre-computed sine and cosine lookup tables to avoid expensive floating-point calculations within game loops.

## Lifecycle & Ownership
- **Creation:** The CircleIterator is instantiated directly via its constructor (`new CircleIterator(...)`) by any client system. It is not managed by a central registry or dependency injection container.
- **Scope:** This is an ephemeral, short-lived object. Its lifetime is typically confined to the scope of the loop that consumes it.
- **Destruction:** The object becomes eligible for garbage collection as soon as all references to it are dropped, which usually occurs immediately after its consuming loop completes. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The CircleIterator is a stateful and **mutable** object. Its internal state, specifically the pointIndex, is advanced on every call to the next method. Consequently, an instance of this iterator is single-use and cannot be reset or reused once exhausted. The initial parameters (origin, radius, pointTotal) are immutable after construction.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access and must be confined to the thread that created it, typically the main game thread. Sharing an instance across multiple threads will result in race conditions and undefined behavior. Do not use this iterator in parallel streams or concurrent collections without external synchronization, which is strongly discouraged.

## API Surface
The public contract is defined by the standard Java Iterator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasNext() | boolean | O(1) | Checks if the iterator has more points to generate. |
| next() | Vector3d | O(1) | Calculates and returns the next point on the circle's circumference. **Warning:** This method allocates a new Vector3d object on each invocation. In performance-critical code, this may create GC pressure. |

## Integration Patterns

### Standard Usage
The CircleIterator is intended to be used in a standard `while` loop. Create a new instance for each distinct operation.

```java
// Example: Spawning 12 pillars in a circle around a central point.
Vector3d centerPoint = new Vector3d(100, 64, 100);
double placementRadius = 15.0;
int pillarCount = 12;

CircleIterator iterator = new CircleIterator(centerPoint, placementRadius, pillarCount);

while (iterator.hasNext()) {
    Vector3d pillarPosition = iterator.next();
    // The Y-coordinate of pillarPosition will match centerPoint.
    world.placeBlock(pillarPosition, Block.PILLAR);
}
```

### Anti-Patterns (Do NOT do this)
- **Iterator Reuse:** Do not attempt to reuse an iterator after it has been fully consumed. The `hasNext` method will return false, and subsequent calls to `next` will produce incorrect results. Always create a new instance.
- **Concurrent Access:** Do not pass an instance of CircleIterator to another thread or use it in a parallel computation. The internal state is not protected by locks.
- **Skipping hasNext:** While the current implementation might not throw a NoSuchElementException, relying on this behavior is unsafe. The contract of an Iterator requires checking `hasNext` before calling `next`. Failure to do so can lead to subtle bugs if the implementation changes.

## Data Pipeline
The CircleIterator acts as a **data source** or **generator**. It does not process incoming data; it creates new data for other systems to consume.

> Flow:
> **CircleIterator** -> (new Vector3d) -> Game Logic (e.g., World Generation, Entity Spawner, VFX System)

