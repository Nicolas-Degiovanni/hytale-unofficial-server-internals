---
description: Architectural reference for Rotation
---

# Rotation

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Utility / Value Type

## Definition
```java
// Signature
public enum Rotation implements NetworkSerializable<com.hypixel.hytale.protocol.Rotation> {
```

## Architecture & Concepts

The Rotation enum is a fundamental, immutable value type that represents discrete, 90-degree rotations around the primary axes. It is a cornerstone of the engine's spatial and geometric calculation system, providing a safe, performant, and predictable mechanism for orienting game objects such as blocks and entities.

Its primary architectural function is to centralize all cardinal rotation logic. By encapsulating the mathematics of 90-degree transformations, it prevents the proliferation of scattered, error-prone, and inconsistent matrix or quaternion calculations throughout the codebase.

As an implementation of NetworkSerializable, Rotation is a first-class citizen in the client-server data model. It has a direct, one-to-one mapping with its network protocol counterpart, `com.hypixel.hytale.protocol.Rotation`. The static CODEC field ensures seamless integration with the engine's serialization framework, allowing world data and entity states to be efficiently encoded and decoded without custom logic.

## Lifecycle & Ownership

-   **Creation:** The four canonical instances—None, Ninety, OneEighty, and TwoSeventy—are instantiated by the Java Virtual Machine during class loading. They are compile-time constants and no other instances can be created.
-   **Scope:** Instances are static and globally accessible, persisting for the entire lifetime of the application. They are effectively permanent, application-scoped constants.
-   **Destruction:** The enum instances are garbage collected only when the JVM shuts down. There is no concept of manual destruction or resource cleanup.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields are final and assigned upon creation. Methods that appear to modify the rotation, such as `add` or `flip`, do not alter the state of the current instance. Instead, they return one of the other pre-existing canonical instances. This design guarantees that a reference to Rotation.Ninety will always represent exactly 90 degrees.

-   **Thread Safety:** **Inherently thread-safe**. Due to its complete immutability, a Rotation instance can be safely shared, passed, and read by any number of threads without locks, synchronization, or risk of data corruption. This makes it highly suitable for use in multi-threaded contexts, such as parallel world generation or physics calculations.

## API Surface

The public API provides a comprehensive toolkit for rotational mathematics, focusing on performance and type safety.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ofDegrees(int degrees) | static Rotation | O(1) | **Strict Factory.** Returns the exact Rotation for 0, 90, 180, or 270 degrees. Throws IllegalArgumentException for any other value. |
| closestOfDegrees(float degrees) | static Rotation | O(1) | **Lenient Factory.** Returns the closest matching Rotation instance, ideal for converting continuous values like player look angles. |
| add(Rotation other) | Rotation | O(1) | Performs modular addition of two rotations and returns the resulting canonical instance. Does not create new objects. |
| rotateY(Vector3i in, Vector3i out) | Vector3i | O(1) | Applies a yaw rotation to the *in* vector, writing the result to the *out* vector. This pattern avoids heap allocations. |
| toPacket() | com.hypixel.hytale.protocol.Rotation | O(1) | Converts the server-side enum to its network-serializable protocol equivalent. |
| rotate(Vector3i vec, Rotation yaw, Rotation pitch) | static Vector3i | O(1) | Utility to apply a sequence of rotations. **Warning:** This method allocates a new vector via clone(). |

## Integration Patterns

### Standard Usage

The primary use case is applying a known rotation to a spatial vector. To maximize performance and minimize garbage collection, it is standard practice to provide a pre-allocated output vector.

```java
// Obtain a rotation value, typically from block state or entity data
Rotation blockYaw = Rotation.Ninety;

// Define an initial offset or position
Vector3i originalOffset = new Vector3i(5, 0, 0);

// Pre-allocate a vector to store the result
Vector3i rotatedOffset = new Vector3i();

// Apply the rotation. The result is written to rotatedOffset.
blockYaw.rotateY(originalOffset, rotatedOffset);

// At this point, rotatedOffset now holds (0, 0, -5)
```

### Anti-Patterns (Do NOT do this)

-   **Invalid Degree Instantiation:** Do not call `ofDegrees()` with arbitrary integer values. It is not a general-purpose conversion function and will throw an exception for any value that is not a multiple of 90. For converting player input or other floating-point data, always use `closestOfDegrees()`.
-   **Incorrect Axis Application:** Applying a `rotateX` (pitch) when a `rotateY` (yaw) is intended is a common source of geometric bugs. Ensure you are using the correct rotation method for the desired spatial transformation.
-   **Enum Identity Comparison:** For performance and correctness, always compare enum instances using the identity operator `==` instead of the `equals()` method. The JVM guarantees that only one instance of each enum constant exists.

    ```java
    // BAD
    if (myRotation.equals(Rotation.None)) { ... }

    // GOOD
    if (myRotation == Rotation.None) { ... }
    ```

## Data Pipeline

Rotation is not a data processing component itself, but it is a critical piece of data that flows through serialization and game logic pipelines.

> **Flow (Serialization):**
> Game State (e.g., Block with **Rotation.Ninety**) → `toPacket()` → `protocol.Rotation.Ninety` → Network Encoder → Network Packet

> **Flow (Deserialization):**
> Network Packet → Network Decoder → `protocol.Rotation.Ninety` → `Rotation.valueOf(packet)` → Game State (e.g., Block with **Rotation.Ninety**)

