---
description: Architectural reference for Vector3i
---

# Vector3i

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class Vector3i {
```

## Architecture & Concepts
Vector3i is a fundamental data structure representing a point or vector in 3D integer space. It is one of the most frequently instantiated objects in the engine, forming the basis for coordinates, offsets, and directional calculations in systems like world generation, physics, AI, and rendering.

The core architectural decision for this class is its **mutability**. Unlike immutable vector classes which return a new instance for each operation, Vector3i methods typically modify the object's internal state directly and return a reference to `this`. This design prioritizes performance by minimizing object allocations and reducing pressure on the garbage collector, which is critical in a high-performance game loop.

Integration with the engine's serialization framework is provided by the static `CODEC` field. This allows Vector3i to be seamlessly serialized to and deserialized from data files (e.g., JSON, binary formats), making it a foundational component for defining prefabs, storing world data, and configuring game assets.

The class also provides a suite of pre-allocated static constants (e.g., ZERO, UP, NORTH) for common vectors. This is an optimization pattern to prevent the repeated creation of identical, frequently used vector objects.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand throughout the codebase, typically as short-lived objects within a method's scope. Common creation patterns include direct instantiation with `new Vector3i()`, cloning via the `clone()` method, or deserialization through the Hytale `Codec` system. Static helper methods like `add` and `max` also produce new instances.
-   **Scope:** The lifetime of a Vector3i instance is generally transient and confined to a single frame or operation. However, instances can be stored as part of a component's state (e.g., an entity's block position), in which case their lifetime is tied to that of the owning component. The static constants (UP, DOWN, etc.) are singletons that persist for the entire application lifetime.
-   **Destruction:** Management is handled entirely by the Java Garbage Collector. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The object's state consists of three public integer fields: `x`, `y`, and `z`. It also contains a private transient `hash` field used to cache the result of `hashCode()`. The state is fully **mutable**, and most methods are designed to alter these fields in-place.
-   **Thread Safety:** This class is **not thread-safe**. The public, mutable fields and lack of internal synchronization make it highly susceptible to race conditions and inconsistent state if an instance is shared and modified across multiple threads without external locking. Its design implicitly assumes single-threaded access, such as within the main game update loop.

**WARNING:** Never share a Vector3i instance between threads without implementing your own synchronization mechanism. Concurrent modifications will lead to unpredictable behavior and data corruption.

## API Surface
The API is designed for chained method calls. Most mathematical operations modify the instance and return `this`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(x, y, z) | Vector3i | O(1) | Sets the vector's components. Resets the hash cache. |
| add(v) | Vector3i | O(1) | Performs in-place vector addition. |
| subtract(v) | Vector3i | O(1) | Performs in-place vector subtraction. |
| scale(s) | Vector3i | O(1) | Performs in-place scalar multiplication. |
| cross(v) | Vector3i | O(1) | **Returns a new Vector3i** instance representing the cross product. Does not modify the original. |
| dot(other) | int | O(1) | Calculates the dot product between this and another vector. |
| distanceSquaredTo(v) | int | O(1) | Calculates the squared Euclidean distance. Prefer this over distanceTo for performance. |
| clone() | Vector3i | O(1) | Creates and returns a new Vector3i instance with the same component values. |
| hashCode() | int | O(1) | Calculates the hash code. The result is cached after the first call. |

## Integration Patterns

### Standard Usage
The class is designed for fluent, chained operations. A temporary vector is often created, manipulated, and then used for a specific calculation before being discarded.

```java
// Example: Calculate a target position
Vector3i playerPosition = new Vector3i(100, 64, 250);
Vector3i offset = new Vector3i(Vector3i.FORWARD); // Start with a forward direction

// Chain operations to modify the offset
offset.scale(10).add(Vector3i.UP);

// Add the final offset to the player's position
playerPosition.add(offset);

// playerPosition is now (100, 65, 240)
```

### Anti-Patterns (Do NOT do this)
-   **Unintended State Modification:** Passing a Vector3i to a method can result in unexpected side effects if the callee modifies the object. The caller's original vector will be altered.

    ```java
    // BAD: The original 'start' vector is modified by the calculateEnd method.
    Vector3i start = new Vector3i(10, 10, 10);
    Vector3i end = calculateEnd(start); // This method might call start.add(...)

    // To prevent this, pass a copy:
    Vector3i end = calculateEnd(start.clone());
    ```

-   **Concurrent Access:** As stated previously, accessing a single Vector3i instance from multiple threads without external locks is unsafe and will lead to data corruption.

-   **Confusion between In-Place and New Instance Methods:** Most methods modify the instance. However, static helpers (`add`, `min`, `max`) and `cross` create new instances. Developers must be aware of this distinction to avoid bugs.

## Data Pipeline
Vector3i is not a processing component itself, but rather the data payload that flows through many engine pipelines. Its primary role in data flow is during serialization and deserialization of game data.

> **Serialization Flow:**
> Game State (e.g., Block Position) -> **Vector3i** -> Hytale Codec System -> JSON or Binary Representation

> **Deserialization Flow:**
> JSON or Binary File -> Hytale Codec System -> **Vector3i** instance -> Component State Initialization

