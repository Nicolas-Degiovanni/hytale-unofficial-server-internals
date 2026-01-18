---
description: Architectural reference for the Shape2D interface, the foundational contract for all 2D geometric forms in the engine.
---

# Shape2D

**Package:** com.hypixel.hytale.math.shape
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Shape2D {
```

## Architecture & Concepts
The Shape2D interface is a foundational contract within the core mathematical and physics engine. It establishes a standardized, polymorphic API for all two-dimensional geometric representations, such as boxes, circles, and complex polygons.

This abstraction is critical for decoupling high-level systems from concrete geometric implementations. Systems like collision detection, physics simulation, spatial partitioning, and user interface hit-testing can operate on any object that fulfills the Shape2D contract. This allows for significant flexibility and extensibility; new shape types can be introduced without requiring modifications to the systems that consume them.

At its core, Shape2D provides two fundamental capabilities:
1.  **Bounding Box Calculation:** The ability to compute an axis-aligned bounding box (AABB) via the `getBox` method. This is essential for broad-phase collision detection and culling optimizations, allowing for fast rejection tests before more expensive, precise checks are performed.
2.  **Point Containment:** The ability to determine if a specific point in 2D space lies within the shape's boundary via the `containsPosition` method. This is the basis for precise hit detection and spatial queries.

## Lifecycle & Ownership
As an interface, Shape2D itself has no lifecycle. The lifecycle of an object implementing Shape2D is entirely determined by its concrete type and its owner.

-   **Creation:** Concrete implementations (e.g., Box2D, Circle2D) are typically instantiated as value objects or as components of a larger entity. For example, a `CollisionComponent` might create and hold a reference to a Shape2D implementation.
-   **Scope:** The scope varies. A Shape2D used for a temporary query might be stack-allocated and live for a single frame. A Shape2D defining a character's hitbox will live as long as the character entity exists.
-   **Destruction:** Managed by the Java Garbage Collector. Ownership is held by the containing object (e.g., a component or system), and the shape is eligible for collection when its owner is destroyed and all references are released.

**WARNING:** Implementations of Shape2D are often intended to be treated as value objects. Avoid retaining long-lived references to temporary shapes generated during collision checks to prevent memory leaks.

## Internal State & Concurrency
The Shape2D interface itself is stateless. However, any class that implements it will inherently contain state defining its geometry (e.g., radius, vertices, dimensions).

-   **State:** Implementations are expected to be stateful. The contract strongly implies that implementations should be **immutable** or treated as such after creation. Modifying a shape's geometry at runtime can lead to severe and difficult-to-debug race conditions, especially between the physics and rendering threads.
-   **Thread Safety:** The interface contract is implicitly **thread-safe for read operations**. All methods are pure functions of their inputs and the shape's internal (and assumed immutable) state. It is expected that any Shape2D can be safely passed between threads for concurrent read-only queries, such as collision checks on a worker thread while the main thread proceeds. Writing to a Shape2D implementation from multiple threads is not supported and will result in undefined behavior.

## API Surface
The public contract is minimal, focusing on essential geometric queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBox(position) | Box2D | Varies | Computes the world-space Axis-Aligned Bounding Box (AABB) for this shape at a given position. Complexity depends on the concrete shape; O(1) for a Box2D, O(N) for a polygon with N vertices. |
| containsPosition(shapePosition, testPosition) | boolean | Varies | Performs a point-in-shape test. Determines if the `testPosition` is inside the shape, assuming the shape is located at `shapePosition`. This is the primary method for precise hit detection. |

## Integration Patterns

### Standard Usage
Shape2D is almost always consumed by systems that need to perform spatial reasoning. The typical pattern involves retrieving a shape from a component and using it for collision or containment tests.

```java
// A simplified collision system checks for an overlap
public void checkCollision(Entity entityA, Entity entityB) {
    CollisionComponent compA = entityA.getComponent(CollisionComponent.class);
    CollisionComponent compB = entityB.getComponent(CollisionComponent.class);

    if (compA == null || compB == null) {
        return; // No shape to test
    }

    Shape2D shapeA = compA.getShape();
    Vector2d posA = entityA.getPosition();

    Shape2D shapeB = compB.getShape();
    Vector2d posB = entityB.getPosition();

    // 1. Broad-phase check using the AABB
    Box2D boxA = shapeA.getBox(posA);
    Box2D boxB = shapeB.getBox(posB);

    if (boxA.intersects(boxB)) {
        // 2. A more precise check would follow here, potentially using
        // containsPosition or a shape-specific intersection method.
        System.out.println("Potential collision detected!");
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Type Casting:** Do not cast a Shape2D to a concrete type to access specific methods. The purpose of the interface is to provide a uniform API. If you need shape-specific logic, use a proper pattern like the Visitor pattern.
    ```java
    // BAD: Defeats the purpose of the interface
    if (shape instanceof Box2D) {
        Box2D box = (Box2D) shape;
        // ... box-specific logic
    }
    ```
-   **State Mutation:** Do not acquire a Shape2D object and attempt to modify its internal geometry. This violates the implicit immutability contract and can corrupt the state of physics and rendering systems that may hold a reference to the same instance.

## Data Pipeline
Shape2D is a data structure, not a processing node. It serves as a source of geometric information for various engine pipelines.

> **Collision Detection Pipeline:**
> Entity Position + **Shape2D Component** -> Collision System -> `getBox()` -> Broad-Phase Check -> `containsPosition()` -> Collision Event Generation

> **Rendering Culling Pipeline:**
> Camera Frustum + Entity Position + **Shape2D Component** -> Culling System -> `getBox()` -> Frustum Intersection Test -> Render Command Generation

