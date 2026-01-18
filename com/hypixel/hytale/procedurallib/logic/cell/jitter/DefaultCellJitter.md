---
description: Architectural reference for DefaultCellJitter
---

# DefaultCellJitter

**Package:** com.hypixel.hytale.procedurallib.logic.cell.jitter
**Type:** Singleton

## Definition
```java
// Signature
public class DefaultCellJitter implements CellJitter {
```

## Architecture & Concepts
The DefaultCellJitter is a foundational, stateless implementation of the CellJitter strategy interface within the procedural generation library. Its primary architectural role is to provide a baseline coordinate transformation from a discrete grid cell space to a continuous world space.

In procedural algorithms like Worley noise or point distribution, a logical grid divides the world. Each grid cell contains one or more feature points. The position of a point is defined by its host cell's integer coordinates (e.g., cx, cy) plus a fractional offset or "jitter" vector within that cell.

This class represents the most direct and simple transformation strategy: it applies the provided offset vector to the cell coordinate without any internal modification, randomness, or scaling. It effectively defines a unit cube (1.0x1.0x1.0) as the boundary for jittering within any given cell. It serves as a default or a "no-op" jittering behavior for algorithms that calculate their own precise offsets and do not require the jittering system to introduce its own randomness.

## Lifecycle & Ownership
- **Creation:** A public static final instance, DEFAULT_ONE, is instantiated by the JVM during class loading. This is the canonical instance and is intended for global use.
- **Scope:** As a static singleton, its lifetime is tied to the application's lifetime. It persists from class loading until the application terminates.
- **Destruction:** The object is eligible for garbage collection only when its class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** DefaultCellJitter is completely stateless and immutable. It contains no instance fields and its behavior is determined exclusively by its method arguments.
- **Thread Safety:** This class is inherently thread-safe. Its methods are pure functions. A single instance, such as DEFAULT_ONE, can be safely shared and invoked by any number of concurrent threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract is defined by the CellJitter interface. All methods are non-blocking and computationally trivial.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaxX() | double | O(1) | Returns the maximum allowed jitter offset on the X-axis, always 1.0. |
| getMaxY() | double | O(1) | Returns the maximum allowed jitter offset on the Y-axis, always 1.0. |
| getMaxZ() | double | O(1) | Returns the maximum allowed jitter offset on the Z-axis, always 1.0. |
| getPointX(int, Double2) | double | O(1) | Calculates the final X coordinate by adding the vector's x component to the cell's X index. |
| getPointY(int, Double2) | double | O(1) | Calculates the final Y coordinate by adding the vector's y component to the cell's Y index. |
| getPointX(int, Double3) | double | O(1) | Calculates the final X coordinate by adding the vector's x component to the cell's X index. |
| getPointY(int, Double3) | double | O(1) | Calculates the final Y coordinate by adding the vector's y component to the cell's Y index. |
| getPointZ(int, Double3) | double | O(1) | Calculates the final Z coordinate by adding the vector's z component to the cell's Z index. |

## Integration Patterns

### Standard Usage
Always use the provided static singleton instance. This avoids unnecessary object allocation for a stateless utility. Procedural generation systems should inject or reference this instance as the default implementation for coordinate transformation.

```java
// Correctly use the shared, canonical instance
CellJitter jitter = DefaultCellJitter.DEFAULT_ONE;

// A pre-calculated offset vector for a point within a cell
DoubleArray.Double2 offset = ...;

// Transform the cell coordinate (15, 30) to a world coordinate
double worldX = jitter.getPointX(15, offset);
double worldY = jitter.getPointY(30, offset);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create new instances of DefaultCellJitter. The class is stateless, and creating new objects provides no benefit while incurring minor overhead from the garbage collector.

    ```java
    // AVOID: Unnecessary object creation
    CellJitter jitter = new DefaultCellJitter();
    ```

## Data Pipeline
This component acts as a simple, pure transformation function within a larger data pipeline. It does not initiate data flow but is a critical step in converting logical grid positions into final world coordinates.

> Flow:
> Procedural Algorithm (e.g., Noise Generator) -> (Integer Cell Coordinate, DoubleN Offset Vector) -> **DefaultCellJitter** -> Final double World Coordinate -> Subsequent System (e.g., Voxel Placer, Mesh Generator)

