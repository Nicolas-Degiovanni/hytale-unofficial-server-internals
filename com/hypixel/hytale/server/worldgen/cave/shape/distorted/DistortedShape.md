---
description: Architectural reference for DistortedShape
---

# DistortedShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface DistortedShape extends IWorldBounds {

   // Nested Factory Interface
   public interface Factory {
      // ... factory methods
   }
}
```

## Architecture & Concepts
The DistortedShape interface is a core contract within the server-side procedural world generation system, specifically for cave carving. It defines a mathematical, volumetric representation of a cave segment, such as a tunnel or a small chamber. This is not a game entity or a rendered object; it is a short-lived, computational primitive used to define a volume in 3D space that should be carved out of the world's block grid.

Unlike simple geometric shapes like cylinders or spheres, a DistortedShape represents an organic, non-uniform volume. It is conceptually defined by a central spline or axis (from a start to an end point) with a variable width and height profile along its length. The "distortion" aspect is critical; implementations are expected to incorporate procedural noise and other transformations to create natural-looking, irregular cave walls, floors, and ceilings.

This interface extends IWorldBounds, which provides an axis-aligned bounding box for the shape. This is a vital optimization, allowing the cave carving system to quickly discard entire world chunks that do not intersect with the shape, avoiding costly per-block calculations.

## Lifecycle & Ownership
- **Creation:** DistortedShape instances are never created directly. They are exclusively instantiated via an implementation of the nested DistortedShape.Factory interface. This factory is typically invoked by a higher-level orchestrator, such as a CaveSystemGenerator, which provides the necessary parameters like origin, direction, length, and size profiles.

- **Scope:** The lifetime of a DistortedShape object is extremely brief and localized. It exists only for the duration of a single cave carving operation. It is a transient object holding the mathematical definition of a cave segment while the generator modifies the world's block data.

- **Destruction:** The object is managed by the Java Garbage Collector. Once the carving algorithm has finished using the shape to define the volume, all references to it are dropped, and it is reclaimed during the next GC cycle. There is no manual cleanup or destruction protocol.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any concrete implementation of DistortedShape is expected to be **effectively immutable** after its creation via the factory. Its internal state consists of the geometric parameters passed during construction (start/end vectors, width/height values, interpolation functions). It does not cache world data or any other external state.

- **Thread Safety:** Implementations are designed to be **thread-safe for read operations**. The world generation process is heavily multi-threaded, with different threads carving different parts of the world. A single DistortedShape instance can be safely queried by multiple worker threads simultaneously to determine its bounds or geometry at a specific point. The immutability of its internal state is the key enabler of this concurrency.

## API Surface
The public contract is focused on querying the geometric properties of the shape.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStart() | Vector3d | O(1) | Returns the starting point of the shape's central axis. |
| getEnd() | Vector3d | O(1) | Returns the ending point of the shape's central axis. |
| getWidthAt(double) | double | O(1) | Calculates the base width of the shape at a given distance along its axis. |
| getHeightAt(double) | double | O(1) | Calculates the base height of the shape at a given distance along its axis. |
| getHeightAtProjection(...) | double | O(N) | **Core Method.** Calculates the final, distorted ceiling height at a world coordinate, applying procedural noise and other effects based on the CaveType and ShapeDistortion parameters. |
| getCeiling(...) | double | O(N) | Convenience method to calculate the final Y-coordinate of the cave roof at a given (x, z) position. |
| getFloor(...) | double | O(N) | Convenience method to calculate the final Y-coordinate of the cave floor at a given (x, z) position. |
| Factory.create(...) | DistortedShape | O(1) | Instantiates a concrete implementation of the shape with specified geometric parameters. |

## Integration Patterns

### Standard Usage
The primary consumer of this interface is a cave carving algorithm. The algorithm obtains a factory, creates a shape instance to represent a cave tunnel, and then uses it to determine which blocks to remove from the world grid.

```java
// A conceptual cave carver using the DistortedShape
void carveCaveSegment(DistortedShape.Factory shapeFactory, WorldGrid world) {
    // 1. Define the cave segment's properties
    Vector3d start = new Vector3d(100, 50, 200);
    Vector3d direction = new Vector3d(0.8, 0.1, 0.6).normalize();
    double length = 50.0;
    double width = 8.0;
    double height = 6.0;

    // 2. Create the shape instance via the factory
    DistortedShape caveTunnel = shapeFactory.create(start, direction, length, width, height);

    // 3. Iterate over blocks within the shape's bounding box
    for (int x = caveTunnel.getLowBoundX(); x < caveTunnel.getHighBoundX(); x++) {
        for (int z = caveTunnel.getLowBoundZ(); z < caveTunnel.getHighBoundZ(); z++) {
            // 4. Query the shape for floor and ceiling height
            double floorY = caveTunnel.getFloor(x, z, ...);
            double ceilingY = caveTunnel.getCeiling(x, z, ...);

            // 5. Modify the world grid
            for (int y = floorY; y < ceilingY; y++) {
                world.setBlock(x, y, z, Block.AIR);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Implementing the Interface Manually:** Do not create your own custom class that implements DistortedShape for a one-off purpose. The world generation system relies on specific behaviors from the standard factory implementations. Always use a provided DistortedShape.Factory.

- **Long-Term Storage:** Do not hold references to DistortedShape objects beyond the scope of a single carving operation. They are not designed for serialization or to be part of a persistent game state. Doing so would result in a significant memory leak.

- **Assuming Simplicity:** Do not assume the shape is a perfect cylinder or capsule. The entire purpose of the interface is to abstract away complex procedural distortion. Rely only on the provided API, such as getFloor and getCeiling, to determine inclusion within the volume.

## Data Pipeline
DistortedShape acts as a function in the data pipeline of world generation. It transforms a set of high-level parameters into a concrete, queryable 3D volume.

> Flow:
> Cave System Parameters (e.g., start point, length, size) -> **DistortedShape.Factory** -> Creates **DistortedShape Instance** -> Cave Carver queries instance for bounds and geometry -> Block-level modifications to World Grid

