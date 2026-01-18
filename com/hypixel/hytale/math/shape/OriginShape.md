---
description: Architectural reference for OriginShape
---

# OriginShape

**Package:** com.hypixel.hytale.math.shape
**Type:** Decorator / Transient

## Definition
```java
// Signature
public class OriginShape<S extends Shape> implements Shape {
```

## Architecture & Concepts
The OriginShape class is a fundamental component of the geometric calculation system, implementing the **Decorator** design pattern. Its primary function is to decouple a shape's intrinsic geometry from its position in world space. This allows a single, complex shape definition to be reused in multiple locations without re-instantiation, significantly improving memory efficiency.

Architecturally, OriginShape acts as a transparent wrapper or a coordinate system translator. It conforms to the Shape interface, allowing it to be used interchangeably with any other Shape object. When a method like containsPosition is invoked, OriginShape intercepts the call, translates the incoming world-space coordinates into the shape's local coordinate system by subtracting its origin, and then delegates the call to the wrapped shape. The reverse is true for methods that operate from the shape outwards, such as getBox.

This design is critical for systems like collision detection, physics simulation, and spatial queries, where entities are composed of primitive shapes that must be tested against world coordinates.

### Lifecycle & Ownership
-   **Creation:** OriginShape instances are lightweight and intended for direct instantiation via the `new` keyword. They are typically created on-demand by higher-level systems that manage game objects or perform spatial calculations.
-   **Scope:** The lifetime of an OriginShape is tied to its owner. It may be a long-lived component as a field within an Entity class, or it may be a short-lived, temporary object created for a single frame's physics query.
-   **Destruction:** Instances are managed by the Java Garbage Collector and are reclaimed when no longer referenced. There are no native resources or explicit cleanup methods associated with this class.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. While the `origin` field is declared final, the Vector3d object it references is itself mutable. More critically, the wrapped `shape` field is public and can be reassigned at any time after construction. This design prioritizes performance and flexibility over safety.
-   **Thread Safety:** **This class is not thread-safe.** Direct access to its public fields and the potential for the underlying shape to be mutable make it entirely unsuitable for concurrent use without external synchronization. All interactions with an OriginShape instance should be confined to a single thread, such as the main game update loop.

## API Surface
The public API forwards most calls to the wrapped shape, applying a coordinate transformation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| containsPosition(x, y, z) | boolean | O(S) | Checks if a world-space point is inside the shape by translating the point to the shape's local space. Complexity depends on the wrapped shape. |
| getBox(x, y, z) | Box | O(S) | Returns the bounding box of the shape, translated to the specified world-space coordinates plus the origin offset. |
| forEachBlock(...) | boolean | O(S) | Iterates over blocks within the shape's volume, translating the iteration space by the origin offset. |
| expand(radius) | void | O(S) | Expands the *underlying* wrapped shape. This operation does not affect the origin vector. |

## Integration Patterns

### Standard Usage
The canonical use case is to define a shape at the coordinate system origin (0,0,0) and then use OriginShape to place it in the world.

```java
// 1. Define a base shape, such as a Box centered at the origin.
Shape baseBox = new Box(-1, -1, -1, 1, 1, 1);

// 2. Create a position vector for the shape in the world.
Vector3d worldPosition = new Vector3d(10, 50, 20);

// 3. Wrap the base shape with an OriginShape to place it.
Shape positionedBox = new OriginShape<>(worldPosition, baseBox);

// 4. Perform checks using world coordinates.
// This will correctly return true.
boolean isInside = positionedBox.containsPosition(10.5, 50.5, 20.5);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Field Modification:** The public fields `origin` and `shape` are exposed for performance reasons. Modifying them after the object has been passed to other systems can lead to unpredictable behavior and race conditions. Treat the object as immutable after construction.
-   **Using the Default Constructor:** `new OriginShape()` creates an object with a null `shape` field. Any subsequent method call that delegates to the shape will throw a NullPointerException. This constructor should be avoided unless the shape is set immediately in a controlled context.
-   **Concurrent Access:** Never share an OriginShape instance across multiple threads without explicit locking. Its mutable state makes it inherently unsafe for concurrency.

## Data Pipeline
OriginShape functions as a transformation stage in a data flow, converting coordinates between different reference frames.

> Flow:
> World Coordinates -> **OriginShape Method (e.g., containsPosition)** -> (Translation by subtracting origin) -> Local Shape Coordinates -> Wrapped Shape Logic -> Result (e.g., boolean)

