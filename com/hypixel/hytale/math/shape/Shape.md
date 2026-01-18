---
description: Architectural reference for the Shape interface, the core contract for geometric volumes in the world.
---

# Shape

**Package:** com.hypixel.hytale.math.shape
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Shape {
```

## Architecture & Concepts
The Shape interface is a foundational contract within the Hytale math engine. It provides a standardized abstraction for any geometric volume, such as boxes, spheres, or more complex polygons. Its primary role is to decouple high-level game systems (like physics, AI, and world interaction) from the specific mathematical implementations of different shapes.

By conforming to this interface, any geometric object can be queried for its bounding box, tested for point containment, and, most critically, be rasterized into the discrete block grid of the game world. This allows a single physics or query algorithm to operate on a variety of collision volumes without needing to know their underlying type. The design heavily favors functional-style iteration via predicates (e.g., TriIntPredicate) to enable high-performance, allocation-free traversal of world blocks covered by the shape.

## Lifecycle & Ownership
As an interface, Shape itself has no lifecycle. The lifecycle pertains to its concrete implementations (e.g., Box, Sphere).

- **Creation:** Implementations are instantiated on-demand by various systems. For example, a physics component might create a Box shape during entity initialization, or a world query might create a temporary Sphere shape to test an area of effect.
- **Scope:** The lifetime of a Shape implementation is strictly tied to its owner. A Shape representing an entity's collision volume will persist as long as the entity exists. A Shape created for a single, transient query will be eligible for garbage collection as soon as the query completes.
- **Destruction:** There is no manual destruction. Shape objects are managed by the Java Garbage Collector and are reclaimed when they are no longer referenced.

## Internal State & Concurrency
- **State:** The Shape contract is stateless, but its implementations are inherently stateful (e.g., a Sphere holds a radius, a Box holds dimensions). The interface design suggests that implementations should be treated as immutable after creation, with the notable exception of the *expand* method. This mutability should be handled with extreme caution.
- **Thread Safety:** **Not thread-safe.** Shape implementations are designed for use within a single, well-defined thread, typically the main game logic thread. There are no internal locks or synchronization mechanisms. Passing Shape instances between threads or modifying them concurrently will lead to race conditions and undefined behavior.

## API Surface
The public contract focuses on spatial queries and world grid iteration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBox(position) | Box | O(1) | Calculates the minimal Axis-Aligned Bounding Box (AABB) that fully encloses the shape at a given world position. Essential for broad-phase collision detection. |
| containsPosition(origin, position) | boolean | O(1) | Performs a point-in-volume test. Returns true if the target position is inside the shape, relative to the shape's origin. |
| expand(amount) | void | O(1) | Modifies the shape by expanding its boundaries outward. **Warning:** This is a mutating operation and should be used with care. |
| forEachBlock(origin, consumer) | boolean | O(N³) | Iterates over every integer block coordinate that intersects with the shape's volume. This is the primary mechanism for rasterizing geometry into the game world. The complexity is proportional to the volume of the shape. |

## Integration Patterns

### Standard Usage
The most common use case is to perform a spatial query against the world grid. A system creates or retrieves a Shape, positions it in the world, and uses forEachBlock to apply logic to every block it touches.

```java
// Get a shape, for example, from a game entity
Shape explosionVolume = new Sphere(5.0); 
Vector3d centerPoint = entity.getPosition();

// Find all blocks of a certain type within the explosion
World world = context.getService(World.class);
explosionVolume.forEachBlock(centerPoint, (x, y, z) -> {
    Block targetBlock = world.getBlockAt(x, y, z);
    if (targetBlock.isDestructible()) {
        world.setBlockAt(x, y, z, Blocks.AIR);
    }
    return true; // Continue iteration
});
```

### Anti-Patterns (Do NOT do this)
- **Unbounded Iteration:** Calling forEachBlock on an extremely large Shape can stall the game thread. These operations must be used on reasonably sized volumes or executed asynchronously if the scale is very large.
- **Ignoring Epsilon:** The forEachBlock variants that accept an epsilon value are critical for precision. Using the default of 0.0 can cause blocks at the very edge of a shape to be missed due to floating-point inaccuracies. Always provide a small epsilon for queries that require perfect boundary detection.
- **Concurrent Modification:** Do not modify a Shape (e.g., via the expand method) on one thread while another thread is reading from it or iterating over it.

## Data Pipeline
The Shape interface acts as a geometric query engine, transforming a continuous mathematical volume into a discrete set of grid coordinates.

> Flow:
> System Logic -> Creates **Shape** (e.g., Sphere) -> **Shape.forEachBlock**(origin) -> Rasterization Algorithm -> Stream of (x, y, z) block coordinates -> Consumer Predicate -> World Interaction

