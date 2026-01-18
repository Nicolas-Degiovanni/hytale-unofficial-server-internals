---
description: Architectural reference for ResultBuffer
---

# ResultBuffer

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Utility / Static Container

## Definition
```java
// Signature
public class ResultBuffer {
   // Contains static nested classes:
   // Bounds2d
   // ResultBuffer2d
   // ResultBuffer3d
}
```

## Architecture & Concepts

The ResultBuffer class is a performance-critical utility designed to eliminate memory allocations within tight loops of the procedural generation library. It acts as a globally accessible, reusable "scratchpad" for geometric calculations, particularly those involving distance checks and nearest-neighbor searches common in algorithms like Voronoi noise.

By providing pre-allocated, static instances of its nested buffer classes (ResultBuffer2d, ResultBuffer3d), the engine avoids the significant overhead of object creation and subsequent garbage collection during world generation. The core architectural principle is to trade thread safety and encapsulation for raw performance by exposing mutable, global state.

This component is not a service or a manager; it is a low-level, stateful container intended for immediate reuse within a single, synchronous computational block. Its design implies that the calling code is responsible for managing its state, including resetting it before use.

### Lifecycle & Ownership
- **Creation:** The static instances `bounds2d`, `buffer2d`, and `buffer3d` are instantiated by the Java Virtual Machine's class loader when the ResultBuffer class is first referenced.
- **Scope:** Application-wide. Once loaded, these objects persist for the entire lifetime of the application. They are global singletons in effect, though not in pattern.
- **Destruction:** The objects are reclaimed only when the application terminates and the class loader is garbage collected. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The nested objects within ResultBuffer are highly mutable. All data fields are public and are intended for direct, repeated modification. The state is transient and holds no value between distinct procedural calculations.

- **Thread Safety:** **WARNING:** This class is fundamentally **thread-unsafe**. The static fields represent shared, global, mutable state. Concurrent access from multiple threads will result in race conditions, where threads overwrite each other's intermediate results, leading to corrupted generation output and non-deterministic behavior.

    Any system utilizing ResultBuffer **must** ensure that access is synchronized externally or that the procedural generation pipeline is confined to a single thread. Failure to do so will cause severe and difficult-to-diagnose bugs.

## API Surface

The primary API is exposed through the public fields and methods of the nested static classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| **bounds2d** | Bounds2d | O(1) | Global instance for 2D bounding box checks. |
| **buffer2d** | ResultBuffer2d | O(1) | Global instance for 2D nearest-neighbor results. |
| **buffer3d** | ResultBuffer3d | O(1) | Global instance for 3D nearest-neighbor results. |
| Bounds2d.assign(...) | void | O(1) | Sets the min/max coordinates of the bounding box. |
| Bounds2d.contains(x, y) | boolean | O(1) | Checks if a 2D point is within the current bounds. |
| ResultBuffer2d.register(...) | void | O(1) | Compares input with stored distance, replacing if closer. |
| ResultBuffer2d.register2(...) | void | O(1) | Tracks the two closest points, replacing if input is closer. |
| ResultBuffer3d.register(...) | void | O(1) | Compares input with stored distance, replacing if closer. |
| ResultBuffer3d.register2(...) | void | O(1) | Tracks the two closest points, replacing if input is closer. |

## Integration Patterns

### Standard Usage

The intended use is within a single function scope where a nearest-neighbor search is performed. The caller is responsible for initializing the buffer's state (e.g., setting the distance to a maximum value) before the loop and reading the results after.

```java
// Example: Finding the closest point in a set
// WARNING: This code must not be run on multiple threads simultaneously.

// 1. Initialize the buffer state for a new calculation.
ResultBuffer.buffer2d.distance = Double.MAX_VALUE;
ResultBuffer.buffer2d.distance2 = Double.MAX_VALUE;

// 2. Iterate and register candidates. The buffer tracks the best result.
for (Point p : pointCloud) {
    double dist = calculateDistance(p, origin);
    ResultBuffer.buffer2d.register(p.hash, p.ix, p.iy, dist, p.x, p.y);
}

// 3. Read the final result from the global buffer.
int closestHash = ResultBuffer.buffer2d.hash;
double closestX = ResultBuffer.buffer2d.x;
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Never access the static buffer fields from multiple threads without an external locking mechanism. This is the most critical anti-pattern and will lead to data corruption.
- **State Leakage:** Do not assume the buffer is in a clean state. Failure to reset the `distance` field to a large value before a calculation will cause results from the previous operation to corrupt the new one.
- **Direct Instantiation:** Do not create new instances like `new ResultBuffer.ResultBuffer2d()`. This completely defeats the performance optimization purpose of the class by causing a new heap allocation. Always use the provided static instances.

## Data Pipeline

ResultBuffer does not participate in a data pipeline in the traditional sense. Instead, it serves as a temporary, mutable state vessel *within* a single processing stage of a larger pipeline, such as world generation.

> Flow:
> Procedural Algorithm Start -> **Initialize ResultBuffer.buffer2d** -> Loop over candidate points -> **Call register() repeatedly** -> Loop End -> Read final state from **ResultBuffer.buffer2d** -> Generate final output (e.g., block type, biome value)

