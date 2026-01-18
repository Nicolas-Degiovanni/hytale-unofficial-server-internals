---
description: Architectural reference for AbstractDistortedShape
---

# AbstractDistortedShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Base Class

## Definition
```java
// Signature
public abstract class AbstractDistortedShape implements DistortedShape {
```

## Architecture & Concepts
AbstractDistortedShape is a foundational component within the server-side world generation pipeline, specifically for procedural cave and tunnel carving. It does not represent a complete geometric shape itself, but rather provides the essential, non-distorted bounding volume for a more complex shape.

Its primary architectural role is to serve as an optimization primitive. By pre-calculating and storing a coarse, axis-aligned bounding box (AABB), it allows world generation algorithms to perform rapid, low-cost spatial culling. Algorithms can quickly discard entire regions of a world chunk that do not intersect with this bounding box, avoiding expensive, high-precision geometric calculations for blocks that are guaranteed to be outside the final shape.

This class provides two constructor patterns, indicating its use for both volumetric shapes (e.g., spheres, ellipsoids) and linear shapes (e.g., tunnels, capsules). Concrete implementations, which extend this class, are responsible for defining the precise geometry and distance functions within these bounds.

### Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. It is constructed via a `super()` call from a concrete subclass, such as a DistortedEllipsoid or DistortedTunnel. These concrete shapes are typically created on-the-fly by higher-level services like a CaveGenerator or a Carver during a chunk generation pass.
- **Scope:** Instances are highly transient and short-lived. Their lifecycle is bound to a single, specific world generation operation. They exist only to define the parameters of a carving action and are not persisted or referenced after the operation completes.
- **Destruction:** Instances are managed by the Java garbage collector. They become eligible for collection as soon as the generating algorithm that created them finishes its execution and the instance falls out of scope. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields representing the bounding box coordinates are declared `final` and are exclusively set within the constructor. Once an AbstractDistortedShape is created, its spatial bounds cannot be altered. This immutability is a critical design feature for predictability and safety in a multithreaded environment.

- **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable nature, an instance can be safely read by multiple world generation worker threads simultaneously without any risk of data corruption or race conditions. No external locking or synchronization is necessary when accessing its state.

## API Surface
The public contract of this class is minimal, focusing entirely on providing access to the pre-calculated bounding box and a static utility function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLowBoundX() | int | O(1) | Returns the minimum X-coordinate of the bounding box. |
| getHighBoundX() | int | O(1) | Returns the maximum X-coordinate of the bounding box. |
| getLowBoundY() | int | O(1) | Returns the minimum Y-coordinate of the bounding box. |
| getHighBoundY() | int | O(1) | Returns the maximum Y-coordinate of the bounding box. |
| getLowBoundZ() | int | O(1) | Returns the minimum Z-coordinate of the bounding box. |
| getHighBoundZ() | int | O(1) | Returns the maximum Z-coordinate of the bounding box. |
| clampPitch(pitch) | static double | O(1) | Clamps a given pitch angle to a safe, non-vertical range. |

## Integration Patterns

### Standard Usage
This class must be extended. A concrete implementation provides the specific shape logic, while this base class handles the AABB. The primary consumer is a "carver" algorithm that iterates only within the provided bounds.

```java
// A hypothetical concrete shape and carver algorithm
// NOTE: DistortedSphere is a hypothetical subclass for this example.

// 1. A generator creates a concrete shape instance.
Vector3d center = new Vector3d(100, 50, 200);
DistortedShape sphere = new DistortedSphere(center, 16.0);

// 2. A carver algorithm consumes the shape and uses its AABB.
for (int y = sphere.getLowBoundY(); y <= sphere.getHighBoundY(); y++) {
    for (int z = sphere.getLowBoundZ(); z <= sphere.getHighBoundZ(); z++) {
        for (int x = sphere.getLowBoundX(); x <= sphere.getHighBoundX(); x++) {
            // 3. Perform a precise check only for blocks inside the AABB.
            if (sphere.isInside(x, y, z)) {
                chunk.setBlock(x, y, z, Block.AIR);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Bounding Box:** The primary purpose of this class is to provide a coarse bounding box for optimization. Iterating over an entire chunk and calling a shape's `isInside` method for every block defeats the purpose of this class and will cause severe performance degradation.
- **Assuming Precise Bounds:** The AABB is intentionally larger than the geometric shape it contains. Do not rely on the bounding box for precise collision or intersection logic; it is only for broad-phase culling.
- **Attempting to Modify State:** The immutable design is intentional. Do not attempt to use reflection or other means to modify the bounds after instantiation. If a different bounding box is needed, create a new shape instance.

## Data Pipeline
AbstractDistortedShape does not process a stream of data. Instead, it acts as a data object that defines a volume for subsequent processing stages in the world generation pipeline.

> Flow:
> Cave Generation Parameters -> `new ConcreteDistortedShape(params)` -> **AbstractDistortedShape (AABB Calculation)** -> Carver Algorithm (Consumes AABB for iteration) -> `ConcreteDistortedShape.isInside()` Check -> Chunk Block Modification

