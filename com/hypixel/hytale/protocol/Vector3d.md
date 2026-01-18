---
description: Architectural reference for Vector3d
---

# Vector3d

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Vector3d {
```

## Architecture & Concepts
The Vector3d class is a fundamental data structure representing a point or vector in 3D space using double-precision floating-point values. Its primary architectural role is to serve as a Data Transfer Object (DTO) within the network protocol layer.

Located in the `com.hypixel.hytale.protocol` package, its design is heavily optimized for high-performance network I/O with the Netty framework. The class defines a strict, fixed-size binary layout of 24 bytes (3 doubles at 8 bytes each). This fixed-size contract is critical for the protocol's parsers, allowing for deterministic and fast deserialization without needing to read size prefixes or delimiters.

The presence of static serialization and validation methods (`deserialize`, `validateStructure`) alongside instance-based serialization (`serialize`) indicates that this class is a self-contained component of a larger, schema-driven protocol framework. It is not a general-purpose mathematical utility; it is a network-first entity.

## Lifecycle & Ownership
- **Creation:** Vector3d instances are created frequently and through multiple pathways:
    - By the network layer when a packet is decoded, via the static `Vector3d.deserialize` factory method.
    - By game logic to represent new spatial data (e.g., positions, velocities) using `new Vector3d(x, y, z)`.
    - By cloning an existing instance via the `clone()` method or the copy constructor to prevent aliasing issues.

- **Scope:** Instances are extremely short-lived and typically method-scoped. They exist only as long as required for a specific calculation or for the lifetime of a containing network packet object. They are not intended to be stored in long-term caches or collections.

- **Destruction:** Management is handled entirely by the Java Garbage Collector. When an instance is no longer referenced, it becomes eligible for collection. There are no manual resource management requirements.

## Internal State & Concurrency
- **State:** The internal state is fully **mutable**. The public fields `x`, `y`, and `z` can be directly modified from any code that holds a reference to the object. This design prioritizes performance and ease of use over encapsulation, which is a common trade-off for internal DTOs.

- **Thread Safety:** This class is **not thread-safe**. Direct, unsynchronized access to its public fields from multiple threads can lead to race conditions, inconsistent state, and memory visibility issues.

    **WARNING:** Never share a single Vector3d instance between threads where at least one thread may modify it. If a vector's data must be passed to another thread, always create a new copy.

## API Surface
The public API is focused on construction and network serialization. Standard object methods like `equals` and `hashCode` are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Vector3d | O(1) | Constructs a new Vector3d by reading 24 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the vector's 24 bytes of data into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the structure in bytes, which is always 24. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer contains at least 24 readable bytes from the given offset. |

## Integration Patterns

### Standard Usage
The class is intended for direct instantiation in game logic and for use by the protocol framework during I/O operations.

```java
// Standard game logic usage for representing a new position
Vector3d spawnPoint = new Vector3d(100.5, 64.0, -250.75);

// Hypothetical usage within a packet's serialization logic
public void serialize(ByteBuf buffer) {
    // other fields...
    this.position.serialize(buffer);
}

// Hypothetical usage within a packet's deserialization logic
public void deserialize(ByteBuf buffer, int offset) {
    // other fields...
    this.position = Vector3d.deserialize(buffer, someOffset);
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** The most severe anti-pattern is sharing a single instance across multiple systems or threads. This will inevitably lead to unpredictable behavior.

    ```java
    // BAD: Both physics and AI systems have a reference to the SAME object
    Vector3d playerPosition = new Vector3d(10, 0, 10);
    physicsSystem.setTarget(playerPosition);
    aiSystem.setTarget(playerPosition);

    // If physicsSystem modifies playerPosition, aiSystem will be affected unexpectedly.

    // GOOD: Provide each system with its own copy
    Vector3d playerPosition = new Vector3d(10, 0, 10);
    physicsSystem.setTarget(playerPosition.clone());
    aiSystem.setTarget(playerPosition.clone());
    ```

- **Manual Serialization:** Do not attempt to read or write three double values manually from a buffer. This bypasses the defined contract, which guarantees Little Endian byte order (`writeDoubleLE`, `getDoubleLE`) and could lead to cross-platform compatibility issues.

## Data Pipeline
As a data structure, Vector3d does not process data itself but is a payload that flows through other systems.

> **Incoming Data Flow:**
> Netty Channel -> ByteBuf -> Packet Decoder -> **Vector3d.deserialize** -> Game Logic

> **Outgoing Data Flow:**
> Game Logic -> `new Vector3d()` -> Packet Encoder -> **vector.serialize(ByteBuf)** -> Netty Channel

