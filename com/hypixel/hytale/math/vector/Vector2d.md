---
description: Architectural reference for Vector2d
---

# Vector2d

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient Value Object

## Definition
```java
// Signature
public class Vector2d {
```

## Architecture & Concepts
The Vector2d class is a fundamental data structure representing a point or direction in 2D space using double-precision floating-point components. It is a cornerstone of the engine's math library, used extensively in physics, rendering, AI, and UI systems for representing positions, velocities, accelerations, and dimensions.

The core architectural decision for this class is its **in-place mutability**. Unlike immutable vector types, most methods in Vector2d such as *add*, *scale*, and *normalize* modify the internal state of the instance itself and return a reference to `this`. This design pattern, often called a fluent interface, is a deliberate performance optimization. By reusing existing Vector2d objects and avoiding the creation of new ones in tight loops (e.g., per-frame physics updates), the engine significantly reduces pressure on the garbage collector, which is critical for maintaining stable, real-time frame rates.

Vector2d also integrates directly with the engine's serialization framework through its static *CODEC* field. This allows vector data to be seamlessly loaded from and saved to game data files, such as entity prefabs or world definitions.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by any system that requires 2D vector math. There is no central factory or pool; developers are expected to instantiate them directly using `new Vector2d(...)`.
- **Scope:** The lifetime of a Vector2d object is typically short and tied to the scope in which it was created. They are frequently created as temporary variables within a single method call. When used as a field within a component (e.g., a position on a transform), its lifecycle is bound to that of the owning object.
- **Destruction:** Management is handled entirely by the Java Garbage Collector. Once an instance is no longer referenced, it is eligible for collection. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The class is highly **mutable**. Its primary state consists of the public fields *x* and *y*. It also contains a private transient field, *hash*, which serves as a cache for the result of the *hashCode* method. Any operation that modifies the vector's components resets this cache, ensuring correctness while optimizing for repeated use in hash-based collections.

- **Thread Safety:** Vector2d is **not thread-safe**. The combination of public fields and mutating methods makes it inherently unsafe for concurrent access. If a single instance is modified by one thread while being read by another, the results are undefined and may lead to data corruption or exceptions.

    **WARNING:** Never share and mutate a Vector2d instance across threads without explicit external synchronization. The recommended pattern for passing vector data between threads is to create a copy using the *clone* method or the copy constructor.

## API Surface
The public API is designed for high-performance, in-place mathematical operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(Vector2d v) | Vector2d | O(1) | Adds the components of another vector to this one. **Mutates the instance.** |
| subtract(Vector2d v) | Vector2d | O(1) | Subtracts the components of another vector from this one. **Mutates the instance.** |
| scale(double s) | Vector2d | O(1) | Multiplies both components by a scalar value. **Mutates the instance.** |
| normalize() | Vector2d | O(1) | Adjusts the vector to a length of 1.0. **Mutates the instance.** Throws if length is zero. |
| dot(Vector2d other) | double | O(1) | Computes the dot product with another vector. Does not mutate the instance. |
| distanceTo(Vector2d v) | double | O(1) | Calculates the Euclidean distance to another vector. Does not mutate the instance. |
| clone() | Vector2d | O(1) | Creates and returns a new Vector2d instance with the same x and y values. |
| equals(Object o) | boolean | O(1) | Performs a value-based comparison of the vector's components. |

## Integration Patterns

### Standard Usage
The fluent, mutable API is intended to be chained for clean and efficient vector manipulation.

```java
// Create a vector for player velocity
Vector2d velocity = new Vector2d(10.0, 0.0);

// Apply friction and a downward force (gravity) in one chain
velocity.scale(0.98).add(0.0, -0.5);

// Normalize to get the direction of movement
Vector2d direction = velocity.clone().normalize();
```

### Anti-Patterns (Do NOT do this)
- **Unintended Side Effects:** The most common pitfall is passing a vector to a method and not realizing that the method may modify it. This can cause subtle and hard-to-trace bugs.

    ```java
    // DANGEROUS: The player's actual position vector is modified by the utility method.
    Vector2d playerPosition = player.getPosition();
    Vector2d projectedPosition = PathfindingUtil.projectOntoNavmesh(playerPosition);
    // At this point, playerPosition has been changed.

    // SAFE: Pass a copy to prevent modification of the original.
    Vector2d playerPosition = player.getPosition();
    Vector2d projectedPosition = PathfindingUtil.projectOntoNavmesh(playerPosition.clone());
    ```

- **Concurrent Modification:** Sharing a Vector2d instance between threads for modification is unsafe and will lead to race conditions.

- **Reference Comparison:** Using `==` to compare two vectors checks for reference equality, not value equality. Always use the *equals* method for value comparison.

## Data Pipeline
Vector2d is a data container, not a processing node. It represents data that flows through various engine systems.

> **Serialization Flow:**
> Game Data File (JSON) -> **BuilderCodec** -> Deserialization -> **Vector2d Instance** -> Game Component

> **Physics Update Flow:**
> User Input -> Intent System -> Physics Engine (reads old **Vector2d** position, calculates new **Vector2d** velocity, updates position) -> Transform Component Updated

