---
description: Architectural reference for CellJitter
---

# CellJitter

**Package:** com.hypixel.hytale.procedurallib.logic.cell.jitter
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface CellJitter {
    // Methods defining the contract
}
```

## Architecture & Concepts
The CellJitter interface defines a formal contract for applying positional displacement, or "jitter," to points within a logical grid cell. It is a foundational component of the procedural generation library, designed to break up the artificial regularity of grid-based algorithms. By abstracting the jitter logic, systems like Voronoi diagram generators or feature placement algorithms can remain agnostic to the specific displacement strategy being used.

This interface embodies the **Strategy Pattern**. It allows the procedural engine to dynamically select or configure different jittering behaviors—such as constant, random, or noise-based—without altering the core generation algorithms that consume the jittered points. Its primary role is to introduce organic-looking imperfections into procedurally generated content.

## Lifecycle & Ownership
As an interface, CellJitter itself has no lifecycle. The following pertains to its concrete implementations, which are treated as value objects.

-   **Creation:** Instances are exclusively intended to be created via the static factory method `CellJitter.of(x, y, z)`. This factory is a critical performance optimization; it returns a shared, static instance of `DefaultCellJitter` for the common 1.0x1.0x1.0 jitter, preventing heap allocation in the most frequent use case. For any other values, it returns a new `ConstantCellJitter` instance.
-   **Scope:** Transient. A CellJitter object is typically short-lived, created on-demand for a specific generation task and held only for the duration of that operation.
-   **Destruction:** Instances are managed by the Java Garbage Collector and are eligible for collection as soon as the generation algorithm that created them completes and no longer holds a reference.

## Internal State & Concurrency
-   **State:** The interface is stateless. All known engine implementations, such as `ConstantCellJitter`, are **immutable**. They are configured with their jitter parameters upon construction and do not change afterward. This design is deliberate and critical for system stability.
-   **Thread Safety:** All implementations are guaranteed to be thread-safe due to their immutability. This allows a single CellJitter instance to be safely shared across multiple worker threads during parallelized world generation without requiring locks or other synchronization primitives.

**WARNING:** Custom implementations of this interface MUST be immutable and thread-safe. A stateful implementation will introduce severe, difficult-to-debug race conditions in the world generation pipeline.

## API Surface
The public contract is minimal, focusing entirely on retrieving jitter parameters and calculating displaced points.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaxX() | double | O(1) | Returns the maximum jitter magnitude along the X-axis. |
| getMaxY() | double | O(1) | Returns the maximum jitter magnitude along the Y-axis. |
| getMaxZ() | double | O(1) | Returns the maximum jitter magnitude along the Z-axis. |
| getPointX(int, Double2) | double | O(1) | Calculates the final X-coordinate for a 2D point. |
| getPointY(int, Double2) | double | O(1) | Calculates the final Y-coordinate for a 2D point. |
| getPointX(int, Double3) | double | O(1) | Calculates the final X-coordinate for a 3D point. |
| getPointY(int, Double3) | double | O(1) | Calculates the final Y-coordinate for a 3D point. |
| getPointZ(int, Double3) | double | O(1) | Calculates the final Z-coordinate for a 3D point. |
| of(double, double, double) | CellJitter | O(1) | Static factory for acquiring a CellJitter instance. |

## Integration Patterns

### Standard Usage
The intended use is to acquire an instance from the static factory and pass it to a procedural generation algorithm that requires it. The generator then uses the interface to calculate final point positions.

```java
// Correctly obtain a jitter strategy for a generation task
CellJitter jitter = CellJitter.of(0.8, 0.8, 0.8);

// A hypothetical generator uses the strategy to place a feature
Point3D baseLocation = new Point3D(10, 20, 30);
int pointIndex = 42; // A seed or index for deterministic results
Point3D finalLocation = myGenerator.calculateJitteredPosition(baseLocation, pointIndex, jitter);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not directly construct implementations like `new ConstantCellJitter(1.0, 1.0, 1.0)`. This bypasses the static factory's caching mechanism for the default case, leading to unnecessary object allocations and increased garbage collector pressure. Always use `CellJitter.of()`.
-   **Stateful Implementations:** Creating a custom implementation of CellJitter that stores mutable state is a severe violation of the design contract. The procedural engine assumes these objects are immutable and thread-safe.

## Data Pipeline
CellJitter acts as a pure function in the data pipeline, transforming an input point and an index into a displaced output point. It does not initiate I/O, network requests, or events.

> Flow:
> Procedural Generator -> Provides Base Point & Seed/Index -> **CellJitter.getPoint...()** -> Returns Displaced Coordinate -> Used for Feature Placement (e.g., Voronoi site, biome boundary)

