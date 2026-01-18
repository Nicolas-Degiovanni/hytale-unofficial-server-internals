---
description: Architectural reference for Vector3f
---

# Vector3f

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient

## Definition
```java
// Signature
public class Vector3f {
```

## Architecture & Concepts
The Vector3f class is a fundamental mathematical primitive within the Hytale engine, representing a point, direction, or offset in three-dimensional space using single-precision floating-point values. Its design prioritizes raw performance over safety, a common and necessary trade-off in high-performance game development.

The core architectural choice is **mutability**. Unlike immutable vector types that return a new instance for every operation, Vector3f methods typically modify the object's internal state directly. This strategy drastically reduces object allocation and garbage collection pressure during intensive calculations, such as those found in the physics, rendering, and game logic loops.

A key characteristic of this class is its dual-purpose nature. It can represent both Cartesian coordinates (x, y, z) and Euler angles (pitch, yaw, roll). The API provides alias methods like getPitch for getX and getYaw for getY to reflect this dual context. While convenient, this design requires developers to be disciplined about the context in which a Vector3f instance is being used.

The class also provides two distinct serialization schemas via the Hytale Codec system:
1.  **CODEC:** The standard serializer for XYZ spatial data.
2.  **ROTATION:** A specialized serializer for Pitch, Yaw, and Roll angle data.

This separation ensures that serialized data is unambiguous and correctly interpreted by different engine systems, such as entity persistence or animation configuration.

## Lifecycle & Ownership
-   **Creation:** Vector3f instances are created on-demand throughout the codebase. They are not managed by a central service or registry. Instantiation occurs via constructors (`new Vector3f()`), the `clone()` method, or static factory methods like `add(v1, v2)`.
-   **Scope:** The lifetime of a Vector3f is typically ephemeral and confined to a local scope, such as the duration of a single method execution or one pass of the game loop. They are frequently used as temporary variables for intermediate calculations.
-   **Destruction:** Instances are subject to standard Java Garbage Collection. Due to their high frequency of creation, the mutable design is critical for performance, as it encourages the *reuse* of existing instances rather than the creation of new ones for each step of a calculation.

## Internal State & Concurrency
-   **State:** The class is highly **mutable**. Its primary state is composed of three public float fields: `x`, `y`, and `z`. It also contains a private transient integer `hash` which serves as a lazy-initialized cache for the object's hash code. Any operation that modifies the vector's components resets the `hash` field to zero, invalidating the cache.

-   **Thread Safety:** **This class is not thread-safe.** Direct, unsynchronized access to its public fields and the lack of any internal locking mechanisms make it inherently unsafe for use across multiple threads. If an instance of Vector3f is shared between threads, all access must be externally synchronized. Failure to do so will result in data races, state corruption, and unpredictable behavior.

## API Surface
The public API is designed for chained, in-place operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(x, y, z) | Vector3f | O(1) | Sets the vector's components. Returns a reference to itself for chaining. |
| add(v) | Vector3f | O(1) | Performs in-place vector addition. Returns a reference to itself. |
| subtract(v) | Vector3f | O(1) | Performs in-place vector subtraction. Returns a reference to itself. |
| scale(s) | Vector3f | O(1) | Performs in-place scalar multiplication. Returns a reference to itself. |
| normalize() | Vector3f | O(1) | Modifies the vector to have a length of 1. Involves a square root operation. |
| dot(other) | float | O(1) | Calculates the dot product with another vector. Does not modify state. |
| cross(v) | Vector3f | O(1) | **Warning:** Returns a **new** Vector3f instance representing the cross product. |
| hashCode() | int | O(1) | Calculates and caches the hash code. Subsequent calls are simple field reads. |

## Integration Patterns

### Standard Usage
The intended usage pattern involves creating a vector and chaining multiple mutating operations to achieve a final result, minimizing object allocation.

```java
// Example: Update an entity's position based on velocity and delta time
Vector3f position = new Vector3f(100.0f, 50.0f, 20.0f);
Vector3f velocity = new Vector3f(0.0f, 0.0f, 15.5f);
float deltaTime = 0.016f;

// Chainable, in-place modification
position.addScaled(velocity, deltaTime);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never share and modify a Vector3f instance across multiple threads without external locking. This is the most severe anti-pattern and will lead to difficult-to-diagnose bugs.

-   **Unintended State Mutation:** Passing a Vector3f to a method may result in its value being changed. If the original state is needed later, you must pass a copy.
    ```java
    // BAD: The original 'direction' vector is modified by the calculateReflection method
    Vector3f direction = new Vector3f(1, 0, 0);
    Vector3f reflection = calculateReflection(direction, surfaceNormal);
    // 'direction' may no longer be (1, 0, 0) here!

    // GOOD: Pass a defensive copy to protect the original state
    Vector3f direction = new Vector3f(1, 0, 0);
    Vector3f reflection = calculateReflection(direction.clone(), surfaceNormal);
    ```

-   **Ignoring API Return Types:** The `cross(v)` method is an exception to the in-place mutation rule; it returns a new vector. Ignoring this return value makes the call useless. For performance, use the `cross(v, result)` overload to write to a pre-existing vector.

## Data Pipeline
Vector3f is not a pipeline stage itself but is the fundamental data payload that flows through many engine systems. It represents the state that is processed at each stage.

> **Flow 1: Entity Serialization**
> Game Save Data (JSON/Binary) -> `CODEC` Deserializer -> **Vector3f** (Entity Position) -> Game World

> **Flow 2: Physics Update**
> Physics System -> Reads **Vector3f** (Current Position) -> Applies Forces -> Writes **Vector3f** (New Position) -> Renderer

---

