---
description: Architectural reference for Box2D
---

# Box2D

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Box2D implements Shape2D {
```

## Architecture & Concepts
Box2D is a fundamental data structure representing an Axis-Aligned Bounding Box (AABB) in two-dimensional space. It is a core component of the engine's spatial reasoning systems, including physics, collision detection, rendering culls, and UI element placement.

The class is designed as a mutable, high-performance object. Its core state, the **min** and **max** vectors, are exposed as public final fields. This is a deliberate design choice to eliminate method call overhead in performance-critical code, such as the inner loops of the physics engine. While the references to the Vector2d objects cannot be changed, the vectors themselves are mutable.

A critical feature is the static **CODEC** field. This integrates Box2D directly into Hytale's serialization and data-definition framework. It allows designers and developers to define bounding boxes in external data files (e.g., JSON or HOCON for entity prefabs) which are then deserialized into live Box2D instances at runtime.

The API surface is a fluent interface, where mutator methods like *offset* or *union* return the instance itself, enabling expressive and efficient method chaining.

## Lifecycle & Ownership
-   **Creation:** Box2D instances are created on-demand. They are typically instantiated as temporary, stack-allocated variables within a single frame or physics tick for calculations. They are also created by the engine's Codec system when deserializing game data.
-   **Scope:** The lifetime is transient and typically confined to the local scope of the method that created it. They are not managed as persistent services or components.
-   **Destruction:** Instances are managed by the Java Garbage Collector. No manual memory management is required. Due to their frequent creation in tight loops, developers should be mindful of allocation pressure, reusing instances where possible.

## Internal State & Concurrency
-   **State:** The class is highly mutable. Its internal state is defined by the **min** and **max** Vector2d fields. Most methods operate by directly modifying these fields. This design promotes object reuse to minimize garbage collection overhead.
-   **Thread Safety:** **This class is not thread-safe.** Its public mutable fields and lack of internal synchronization make it inherently unsafe for concurrent access. Any attempt to read and write a Box2D instance from different threads simultaneously will lead to race conditions and undefined behavior. All operations on a given instance must be externally synchronized or confined to a single thread, such as the main game loop or a dedicated physics thread.

## API Surface
The public API is designed for performance and chaining.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| union(Box2D bb) | Box2D | O(1) | Expands this box to contain the other box. Modifies the instance. |
| isIntersecting(Box2D other) | boolean | O(1) | Performs a fast AABB intersection check. Does not modify the instance. |
| offset(Vector2d pos) | Box2D | O(1) | Translates the box by the given vector. Modifies the instance. |
| getBox(double x, double y) | Box2D | O(1) | **Creates and returns a new Box2D** instance, offset from this one. |
| containsPosition(origin, pos) | boolean | O(1) | Checks if a world-space position is inside the box relative to an origin. |
| CODEC | BuilderCodec | N/A | Static field used by the engine to serialize and deserialize Box2D instances. |

## Integration Patterns

### Standard Usage
Box2D is commonly used for collision detection logic. Instances are created, manipulated, and then used for intersection tests within a single logical update.

```java
// Example: Basic collision check during a physics update
Box2D playerHitbox = new Box2D(player.position, player.position.add(1, 2));
Box2D wall = new Box2D(10, 0, 11, 10);

// Check for collision
if (playerHitbox.isIntersecting(wall)) {
    // Resolve collision
}

// Reuse the hitbox for a swept collision check next frame
Vector2d velocity = new Vector2d(0.5, 0);
playerHitbox.sweep(velocity); // Expand the box in the direction of movement
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never share and modify a single Box2D instance across multiple threads without explicit locking. This is the most common source of errors when using this class.
-   **Ignoring Factory Method Returns:** The method *getBox* creates a **new** instance. Ignoring its return value is a logical error, as the original instance is not modified.
-   **Assuming Normalization:** Do not assume the **min** vector's components are always less than the **max** vector's. If manipulating fields directly, you may need to call *normalize* to restore a valid state.

## Data Pipeline
The static CODEC enables Box2D to be a key part of the engine's data pipeline, translating human-readable definitions into in-memory objects.

> Flow:
> Entity Definition File (JSON) -> Engine Codec System -> **Box2D Instance** -> Physics Component -> Collision Detection System

