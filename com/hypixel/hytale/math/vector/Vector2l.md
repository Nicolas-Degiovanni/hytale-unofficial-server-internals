---
description: Architectural reference for Vector2l
---

# Vector2l

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Vector2l {
```

## Architecture & Concepts

The Vector2l class is a fundamental mathematical primitive within the Hytale engine, representing a two-dimensional vector or coordinate using 64-bit long integers. Its primary purpose is to handle discrete, high-precision spatial calculations where floating-point inaccuracies are unacceptable. This makes it essential for core systems like world generation, block and chunk coordinates, and grid-based logic.

The core design philosophy of Vector2l prioritizes performance and memory efficiency over immutability. Most arithmetic and assignment operations are **mutating**; they modify the internal state of the instance directly and return a reference to `this`. This fluent interface allows for chaining operations without generating intermediate object allocations, a critical optimization for the high-frequency calculations common in a game loop.

Integration with Hytale's serialization system is a first-class feature, exposed via the static `CODEC` field. This allows Vector2l instances to be seamlessly encoded to and decoded from various data formats, such as network packets, world save files, and asset definitions.

## Lifecycle & Ownership

-   **Creation:** Vector2l instances are created on-demand throughout the engine. They are considered lightweight, short-lived objects. Common creation points include deserializing game state, calculating entity movement vectors for a single frame, or representing temporary offsets. A pool of static constants (e.g., Vector2l.ZERO, Vector2l.UP) is provided for common, immutable values to prevent unnecessary allocations.
-   **Scope:** The lifetime of a Vector2l instance is typically brief, often confined to the scope of a single method or frame update. However, they are also used as persistent fields within components, such as an entity's chunk position.
-   **Destruction:** Instances are managed by the Java Garbage Collector. Due to the high frequency of instantiation, the mutable design is a deliberate choice to reduce GC pressure.

## Internal State & Concurrency

-   **State:** The class is highly **mutable**. Its primary state consists of the public `x` and `y` long integer fields. It also maintains a private, transient `hash` field which serves as a lazy-initialized cache for its hash code. Any operation that modifies the `x` or `y` coordinates will invalidate this cache by resetting it to zero, triggering a recalculation on the next call to `hashCode`.
-   **Thread Safety:** **This class is not thread-safe.** Its mutable nature, direct field access, and lack of internal synchronization make it inherently unsafe for concurrent use. If a Vector2l must be passed between threads, the sender must either guarantee no further mutations will occur or the receiver must create a defensive copy using the `clone` method.

**WARNING:** Sharing and concurrently modifying a Vector2l instance across multiple threads will lead to race conditions, inconsistent state, and non-deterministic behavior. All multi-threaded access must be protected by external synchronization mechanisms.

## API Surface

The public API is designed for high-performance, chained operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(long x, long y) | Vector2l | O(1) | Mutates the vector, setting its components. Returns `this`. |
| add(Vector2l v) | Vector2l | O(1) | Mutates the vector by adding another. Returns `this`. |
| subtract(Vector2l v) | Vector2l | O(1) | Mutates the vector by subtracting another. Returns `this`. |
| scale(long s) | Vector2l | O(1) | Mutates the vector by scaling its components. Returns `this`. |
| dot(Vector2l other) | long | O(1) | Non-mutating. Calculates the dot product. |
| distanceSquaredTo(Vector2l v) | long | O(1) | Non-mutating. Calculates the squared distance, avoiding a costly square root. |
| clone() | Vector2l | O(1) | Non-mutating. Creates a new Vector2l instance with identical values. |
| hashCode() | int | O(1) | Non-mutating. Returns a cached hash code. The first call computes the hash. |

## Integration Patterns

### Standard Usage

The fluent, mutable API is intended for chained calculations to minimize allocations.

```java
// Create a vector representing a player's base position
Vector2l playerPosition = new Vector2l(100L, 500L);

// Calculate a target position by applying an offset and scaling it
Vector2l offset = new Vector2l(5L, -2L);
playerPosition.add(offset).scale(2L);

// playerPosition is now (210, 496)
```

### Anti-Patterns (Do NOT do this)

-   **Unintended Mutation:** The most common pitfall is assuming arithmetic methods are non-mutating. This can lead to subtle and severe bugs.

    ```java
    // DANGEROUS: This modifies the original 'start' vector.
    Vector2l start = new Vector2l(10, 10);
    Vector2l end = start.add(new Vector2l(5, 5)); // 'start' is now (15, 15)
    
    // CORRECT: Clone the vector first if the original must be preserved.
    Vector2l safeStart = new Vector2l(10, 10);
    Vector2l safeEnd = safeStart.clone().add(new Vector2l(5, 5)); // 'safeStart' remains (10, 10)
    ```

-   **Concurrent Access:** Do not read or write a Vector2l instance from multiple threads without external locking. The cached hash code mechanism is not atomic and can be corrupted.

-   **Manual Serialization:** Do not implement custom serialization logic. The provided static `CODEC` is integrated with the engine's versioning and validation systems and must be used.

## Data Pipeline

Vector2l is not a processing stage in a pipeline but rather a data payload that flows through them. Its most critical role is in the data serialization and deserialization pipeline managed by the Hytale Codec framework.

> Flow:
> Raw Data (JSON, Binary) -> Hytale Codec Framework -> **Vector2l Instance** -> Game System (e.g., Entity Position Component, World Storage)

