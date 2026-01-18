---
description: Architectural reference for Vector3l
---

# Vector3l

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient Value Object

## Definition
```java
// Signature
public class Vector3l {
```

## Architecture & Concepts
The Vector3l class is a fundamental data structure representing a 3-dimensional vector or point using 64-bit long integer components. Its primary role is to define positions, directions, and offsets within the game world, particularly for systems requiring high precision over vast distances, such as world coordinates for blocks and entities.

The core design philosophy of Vector3l prioritizes performance by being **mutable**. Most arithmetic operations like add, subtract, and scale modify the internal state of the instance itself rather than allocating a new object. This design significantly reduces garbage collector pressure in performance-critical loops, such as physics simulations, world generation, and rendering calculations.

A critical architectural feature is the static CODEC field. This integrates Vector3l directly into Hytale's serialization and data definition framework. It allows vector data to be seamlessly read from and written to configuration files, network packets, and save game formats, making it a foundational type for game data.

The class also provides a rich set of static constants (e.g., ZERO, UP, FORWARD) to represent common directional vectors, preventing redundant object creation for frequently used values.

## Lifecycle & Ownership
- **Creation:** Vector3l instances are typically short-lived. They are created in one of three ways:
    1.  **Direct Instantiation:** Via `new Vector3l(x, y, z)`.
    2.  **Cloning:** Using the `clone()` method to create a distinct copy.
    3.  **Deserialization:** Automatically instantiated by the Hytale `Codec` system when loading data from a file or network stream.
- **Scope:** The scope of a Vector3l instance is almost always local to a method or a single frame's calculation. They are passed as parameters, modified, and then typically discarded. Static constants like Vector3l.UP persist for the entire application lifetime.
- **Destruction:** Instances are managed by the Java Garbage Collector. There is no manual cleanup required. Once an instance is no longer referenced, it becomes eligible for collection.

## Internal State & Concurrency
- **State:** The class is **highly mutable**. Its primary state consists of three public long fields: x, y, and z. It also contains a private transient integer field, hash, which serves as a cache for its hash code. Any operation that modifies the vector's components resets this hash to 0, invalidating the cache and forcing a recalculation on the next call to hashCode().

- **Thread Safety:** **WARNING:** Vector3l is **not thread-safe**. Its mutable nature and lack of internal synchronization mechanisms make it inherently unsafe for concurrent modification. If an instance is shared between threads, access must be controlled by external synchronization (e.g., locks). Failure to do so will result in race conditions and unpredictable application behavior. The standard practice is for each thread to work with its own distinct Vector3l instances.

## API Surface
The public API is designed for high-performance, in-place mathematical operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(x, y, z) | Vector3l | O(1) | Resets the vector's components to the provided values. Returns the modified instance. |
| add(v) | Vector3l | O(1) | Performs in-place vector addition. **Warning:** This modifies the current object. |
| subtract(v) | Vector3l | O(1) | Performs in-place vector subtraction. **Warning:** This modifies the current object. |
| scale(s) | Vector3l | O(1) | Performs in-place scalar multiplication. **Warning:** This modifies the current object. |
| dot(other) | long | O(1) | Computes the dot product with another vector. This is a non-mutating operation. |
| distanceSquaredTo(v) | long | O(1) | Calculates the squared Euclidean distance. Prefer this over distanceTo for comparisons to avoid costly square root calculations. |
| clone() | Vector3l | O(1) | Creates and returns a new Vector3l instance with the same component values. |
| hashCode() | int | O(1) amortized | Returns a hash code for the vector. The result is cached until the vector is modified. |

## Integration Patterns

### Standard Usage
The class is designed for fluent, chained operations. Developers should reuse instances where possible to minimize object allocation.

```java
// Example: Calculate a new block position
Vector3l playerPosition = new Vector3l(100L, 64L, -250L);
Vector3l offset = new Vector3l(Vector3l.FORWARD); // Create from constant

// Chain mutable operations on the offset vector
offset.scale(10).add(Vector3l.UP);

// Create a final position by cloning the original and adding the calculated offset
Vector3l targetPosition = playerPosition.clone().add(offset);
```

### Anti-Patterns (Do NOT do this)
- **Unintended Mutation:** The most common error is assuming methods return a new instance. They do not.
    ```java
    // WRONG: This modifies originalPosition
    Vector3l originalPosition = new Vector3l(10, 10, 10);
    Vector3l newPosition = originalPosition.add(Vector3l.RIGHT); 
    // Now originalPosition and newPosition both refer to the same
    // modified object: {x=11, y=10, z=10}

    // CORRECT: Clone before mutation to preserve the original
    Vector3l originalPosition = new Vector3l(10, 10, 10);
    Vector3l newPosition = originalPosition.clone().add(Vector3l.RIGHT);
    ```
- **Cross-Thread Sharing:** Never pass a reference to a Vector3l instance to another thread without ensuring all access to it is synchronized.
    ```java
    // DANGEROUS: Race condition waiting to happen
    Vector3l sharedPosition = new Vector3l();
    
    // Thread 1
    new Thread(() -> sharedPosition.add(1, 0, 0)).start();

    // Thread 2
    new Thread(() -> sharedPosition.add(0, 1, 0)).start();
    ```

## Data Pipeline
Vector3l is a fundamental component of the engine's data serialization pipeline via its static CODEC.

> **Serialization Flow:**
> Game Logic -> **Vector3l Instance** -> `Vector3l.CODEC` -> Serialized Output (e.g., JSON, Binary)

> **Deserialization Flow:**
> Serialized Input (e.g., JSON, Binary) -> `Vector3l.CODEC` -> **Vector3l Instance** -> Game Logic

