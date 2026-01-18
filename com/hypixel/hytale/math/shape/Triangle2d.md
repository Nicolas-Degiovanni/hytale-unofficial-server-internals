---
description: Architectural reference for Triangle2d
---

# Triangle2d

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient

## Definition
```java
// Signature
public class Triangle2d {
```

## Architecture & Concepts
The Triangle2d class is a fundamental geometric primitive within the Hytale math library. It represents a simple polygon defined by three vertices in a two-dimensional Cartesian coordinate system.

This class serves as a foundational data structure for higher-level systems, including:
- **Collision Detection:** Used for precise hit-testing and intersection checks.
- **Mesh Triangulation:** Acts as the basic building block for 2D and 3D procedural mesh generation.
- **Pathfinding & Navigation:** Can be used to define walkable areas or obstacles within a navigation mesh.
- **Rendering:** Forms the basis for rasterization and shader calculations.

It is designed as a lightweight, mutable data container, optimized for high-frequency allocation and short-lived use within tight loops, such as physics steps or rendering frames. It does not contain complex logic, delegating such operations to dedicated utility classes or engine systems.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by any system requiring geometric representation. There is no central factory or manager. It is common to see these objects instantiated directly within method scopes.
- **Scope:** The lifetime of a Triangle2d instance is typically ephemeral. It is intended to exist only for the duration of a specific calculation or algorithm and then be discarded.
- **Destruction:** Management is handled entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods. Once an instance is no longer referenced, it becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The internal state is **mutable** and consists of three Vector2d objects representing the vertices A, B, and C. The public setters (setA, setB, setC) allow for direct modification of the triangle's geometry after instantiation. This mutability is a performance consideration, allowing for the reuse of objects to reduce GC pressure.

- **Thread Safety:** This class is **not thread-safe**. Its methods are non-atomic and contain no internal synchronization mechanisms. Concurrent modification of its vertices from multiple threads will lead to race conditions and unpredictable geometric states.

**WARNING:** Any system that passes Triangle2d instances between threads must implement its own external locking or ensure the object is treated as immutable after being shared.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Triangle2d(a, b, c) | constructor | O(1) | Creates a triangle from three distinct Vector2d points. |
| getMinX() / getMaxX() | double | O(1) | Calculates the minimum or maximum X coordinate among the three vertices. |
| getMinY() / getMaxY() | double | O(1) | Calculates the minimum or maximum Y coordinate among the three vertices. |
| getRandom(random) | Vector2d | O(1) | Generates a new Vector2d containing a random point uniformly distributed within the triangle. |
| getRandom(random, vec) | Vector2d | O(1) | Generates a random point within the triangle and stores it in the provided output vector to prevent allocation. |

## Integration Patterns

### Standard Usage
A Triangle2d should be treated as a temporary, stack-like variable. It is instantiated, used for a calculation, and then its reference is dropped.

```java
// Example: Finding a random spawn point within a triangular zone
Vector2d vert1 = new Vector2d(0, 10);
Vector2d vert2 = new Vector2d(-5, 0);
Vector2d vert3 = new Vector2d(5, 0);

Triangle2d spawnZone = new Triangle2d(vert1, vert2, vert3);
Random rng = new Random();

// Calculate a random point inside the zone
Vector2d spawnPoint = spawnZone.getRandom(rng);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not retain a reference to a Triangle2d instance in a long-lived object and pass it to multiple subsystems that may modify it. This creates implicit dependencies and unpredictable behavior. If a triangle needs to be shared, either create defensive copies or establish a clear contract of immutability.
- **Concurrent Modification:** Never call setter methods (e.g., setA) from one thread while another thread is performing a calculation (e.g., getMinX) on the same instance without external synchronization.

## Data Pipeline
As a data structure, Triangle2d does not process data itself but is rather the payload that flows through various engine pipelines.

> **Typical Flow in Mesh Generation:**
> Point Cloud Data -> Delaunay Triangulation Algorithm -> **Set of Triangle2d objects** -> Mesh Data Structure -> Renderer

> **Typical Flow in Collision Detection:**
> Physics Body A -> Bounding Box Check -> **Triangle2d Intersection Test** -> Collision Event

