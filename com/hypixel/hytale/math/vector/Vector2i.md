---
description: Architectural reference for Vector2i
---

# Vector2i

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient Value Object

## Definition
```java
// Signature
public class Vector2i {
```

## Architecture & Concepts

The Vector2i class is a fundamental data structure representing a 2D vector or point with integer components. It is a cornerstone of the engine's mathematical library, used extensively for grid-based positions, UI coordinates, texture offsets, and directional calculations where floating-point precision is not required.

Architecturally, Vector2i is designed as a **mutable, high-performance object**. Unlike immutable vector types that create a new object for each operation, Vector2i methods modify the internal state of the instance itself and return a reference to `this`. This fluent interface pattern minimizes garbage collection pressure in performance-critical code, such as the main game loop or rendering pipeline, where thousands of vector calculations may occur per frame.

A critical aspect of its design is the integration with the engine's serialization framework via the static CODEC field. This allows Vector2i values to be defined directly in data files (e.g., JSON, HOCON), enabling data-driven configuration of entity positions, UI layouts, and other game parameters.

## Lifecycle & Ownership

-   **Creation:** Instances are created directly via the `new` keyword for transient calculations. The engine's CODEC system also instantiates this class when deserializing game data. Static constants like Vector2i.ZERO or Vector2i.UP are pre-allocated at class-loading time.
-   **Scope:** Vector2i instances are intended to be short-lived. They are typically created as temporary variables within a method's scope, used for a series of calculations, and then discarded. They are not designed to be long-term state holders unless they are part of a larger, well-encapsulated component.
-   **Destruction:** As a standard Java object, instances are managed by the Garbage Collector and are destroyed when they are no longer referenced.

## Internal State & Concurrency

-   **State:** The class is **highly mutable**. Its primary state consists of the public integer fields `x` and `y`. It also contains a private transient `hash` field, which serves as a performance cache for the object's hash code. Any operation that modifies `x` or `y` resets this cache, forcing a recalculation on the next call to hashCode.

-   **Thread Safety:** **This class is not thread-safe.** The public mutable fields and lack of internal synchronization mean that concurrent access from multiple threads will lead to race conditions, inconsistent state, and unpredictable behavior. If a Vector2i instance must be shared between threads, access must be controlled through external synchronization mechanisms.

## API Surface

The public API is designed for chained, in-place mathematical operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(x, y) | Vector2i | O(1) | Sets the vector's components. Returns `this` for chaining. |
| add(v) | Vector2i | O(1) | Performs in-place vector addition. Returns `this`. |
| subtract(v) | Vector2i | O(1) | Performs in-place vector subtraction. Returns `this`. |
| scale(s) | Vector2i | O(1) | Performs in-place scalar multiplication. Returns `this`. |
| dot(other) | int | O(1) | Calculates the dot product with another vector. Does not modify state. |
| distanceSquaredTo(v) | int | O(1) | Calculates the squared Euclidean distance. Avoids a costly square root operation. |
| clone() | Vector2i | O(1) | Creates a new Vector2i instance with the same component values. |
| hashCode() | int | O(1) | Returns a cached or newly computed hash code. |

## Integration Patterns

### Standard Usage

The fluent API is intended for creating clean, allocation-free calculation chains. Always create a new instance or clone an existing one for storing results to avoid side effects.

```java
// Calculate a new position from a base, an offset, and a direction
Vector2i basePosition = new Vector2i(100, 250);
Vector2i offset = new Vector2i(5, 10);
Vector2i direction = Vector2i.UP.clone().scale(50); // Clone static constant before mutation

// The result is calculated in-place within the basePosition instance
basePosition.add(offset).add(direction);

// basePosition is now (105, 310)
```

### Anti-Patterns (Do NOT do this)

-   **Modification of Static Constants:** The static constants (e.g., Vector2i.ZERO, Vector2i.UP) are mutable. Modifying them will cause catastrophic, engine-wide side effects. Always clone them before performing mutating operations.
    ```java
    // CRITICAL ERROR: This modifies the global UP vector for the entire application
    Vector2i.UP.scale(10); 
    ```

-   **Unmanaged State Sharing:** Passing a reference to a Vector2i to multiple systems without defensive cloning can lead to unpredictable state corruption. One system's modification will silently affect the other.
    ```java
    // DANGEROUS: Both player and camera now reference the SAME object
    Vector2i sharedPosition = new Vector2i(10, 10);
    player.setPosition(sharedPosition);
    camera.setTarget(sharedPosition);

    // This moves the player AND unintentionally teleports the camera's target
    player.getPosition().add(5, 0);
    ```

-   **Concurrent Access:** Using a Vector2i instance across multiple threads without explicit locking will result in data corruption.

## Data Pipeline

Vector2i is fully integrated into the engine's data-driven asset pipeline via its static CODEC.

> Flow:
> JSON/HOCON Asset File -> Hytale Codec Deserializer -> **Vector2i Instance** -> Game Component State (e.g., PositionComponent) -> Game Systems (e.g., Physics, Rendering)

