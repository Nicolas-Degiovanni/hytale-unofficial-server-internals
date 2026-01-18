---
description: Architectural reference for Vector3d
---

# Vector3d

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient

## Definition
```java
// Signature
public class Vector3d {
```

## Architecture & Concepts
The Vector3d class is a fundamental mathematical primitive representing a location or direction in three-dimensional space using double-precision floating-point components. It is one of the most frequently instantiated objects in the engine, forming the backbone for physics, rendering, entity positioning, and AI calculations.

A critical design decision for this class is its **mutability**. Most arithmetic and geometric operations modify the instance's internal state directly and return a reference to `this`. This design promotes method chaining and reduces garbage collection pressure by allowing the reuse of vector objects within tight loops, a common scenario in game engine development.

Vector3d is deeply integrated with Hytale's data-driven framework via its static **CODEC** field. This allows for the seamless serialization and deserialization of vector data from game asset files (e.g., JSON definitions for entity properties or model attachments), making it a core component of the engine's content pipeline. The class also provides a rich set of static constants (e.g., **UP**, **FORWARD**, **ZERO**) to represent common directions, preventing unnecessary object allocations for frequently used values.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand throughout the codebase. Common creation points include physics updates, rendering calculations, or deserialization from asset files via the **CODEC**. It is a high-frequency, short-lived object.
- **Scope:** The scope of a Vector3d instance is typically transient, often confined to a single method or a single frame update. They are frequently used as temporary variables for intermediate calculations.
- **Destruction:** Managed entirely by the Java Garbage Collector. Once an instance is no longer referenced, it becomes eligible for collection. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The state is defined by three public `double` fields: **x**, **y**, and **z**. The class is highly **mutable**. A fourth private transient field, **hash**, is used to cache the result of the **hashCode** method. Any operation that modifies the vector's components resets this cache.

- **Thread Safety:** This class is **not thread-safe**. The public mutable fields and lack of any internal synchronization mechanisms make it inherently unsafe for concurrent access. Modifying a Vector3d instance from multiple threads without external locking will result in race conditions, memory visibility issues, and unpredictable behavior. Systems that operate on Vector3d across threads must implement their own synchronization strategies, such as using thread-local instances or explicit locks.

## API Surface
The public API is designed for high-performance, chained operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(x, y, z) | Vector3d | O(1) | Sets the vector's components. Returns `this` for chaining. |
| add(v) | Vector3d | O(1) | Performs an in-place addition. Returns `this`. |
| subtract(v) | Vector3d | O(1) | Performs an in-place subtraction. Returns `this`. |
| scale(s) | Vector3d | O(1) | Performs an in-place scalar multiplication. Returns `this`. |
| normalize() | Vector3d | O(1) | Modifies the vector to have a length of 1. Involves a square root. |
| distanceTo(v) | double | O(1) | Calculates the Euclidean distance to another vector. Does not modify state. |
| cross(v) | Vector3d | O(1) | **Warning:** Returns a **new** Vector3d instance representing the cross product. Does not modify the original. |
| clone() | Vector3d | O(1) | Creates and returns a new Vector3d instance with identical component values. |

## Integration Patterns

### Standard Usage
The fluent, chainable API is the intended pattern for performing sequential calculations without creating intermediate objects.

```java
// Example: Calculate a new position
Vector3d position = new Vector3d(10.0, 20.0, 30.0);
Vector3d velocity = new Vector3d(0.1, 0.0, -0.2);
double deltaTime = 0.016;

// Chain methods to update position in-place
position.addScaled(velocity, deltaTime);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Passing a single Vector3d instance to multiple systems that may modify it is extremely dangerous. This creates implicit dependencies and leads to bugs that are difficult to trace. When passing a vector to another system that might store or modify it, always pass a copy.
  ```java
  // BAD: Both entity and physics system have a reference to the SAME mutable object
  Vector3d sharedPosition = new Vector3d(0, 0, 0);
  player.setPosition(sharedPosition);
  physics.trackObject(sharedPosition);

  // GOOD: Give each system its own copy
  Vector3d initialPosition = new Vector3d(0, 0, 0);
  player.setPosition(initialPosition.clone());
  physics.trackObject(initialPosition.clone());
  ```

- **Cross-Thread Modification:** Never modify a Vector3d instance from one thread while another thread might be reading or writing to it. This will cause data corruption. Use thread-local pools of vectors or other thread-safe mechanisms.

## Data Pipeline
Vector3d is a fundamental data type that flows through nearly all engine systems. It does not process data itself but is the data being processed.

> **Asset Loading Pipeline:**
> Game Asset File (JSON) -> Hytale Codec System -> **Vector3d** (deserialized) -> Entity Component State

> **Physics Update Pipeline:**
> Game Logic (e.g., Player Input) -> Velocity Calculation (uses **Vector3d**) -> Physics Engine -> Position Update (modifies **Vector3d**) -> Renderer

