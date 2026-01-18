---
description: Architectural reference for Vector3i
---

# Vector3i

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Object

## Definition
```java
// Signature
public class Vector3i {
```

## Architecture & Concepts
The Vector3i class is a fundamental data structure representing a 3-dimensional integer coordinate. It is a Plain Old Java Object (POJO) primarily designed for high-performance network communication and in-memory spatial representation. Its location within the *com.hypixel.hytale.protocol* package underscores its role as a Data Transfer Object (DTO) for network packets.

This class is not a managed service; it is a value-type object. Its design prioritizes raw performance and low memory overhead over encapsulation. The public fields (x, y, z) allow for direct, high-speed manipulation, avoiding the overhead of method calls. This pattern is common in performance-critical areas of game engines, such as world manipulation, physics, and rendering, where millions of such objects may be processed per frame.

The static methods like *deserialize* and *validateStructure*, along with the fixed-size constants, indicate that Vector3i is a component of a larger, highly structured network protocol system that relies on predictable, fixed-width data layouts for efficient buffer processing.

## Lifecycle & Ownership
- **Creation:** Vector3i instances are created ephemerally and in high volume. They are instantiated either by the network layer calling the static *Vector3i.deserialize* method when decoding an incoming packet, or directly by game logic via `new Vector3i()` for calculations.
- **Scope:** The object's lifetime is typically very short. It exists for the duration of a single logical operation, such as processing a game event or calculating a block position. It is not registered with or managed by any central system.
- **Destruction:** Ownership is transient. Once an operation completes, the instance is abandoned and becomes eligible for garbage collection. There are no manual cleanup or `close()` methods.

## Internal State & Concurrency
- **State:** The state of Vector3i is **fully mutable**. Its core data—the x, y, and z coordinates—are public integer fields, allowing any external code to modify them directly after instantiation. This is a deliberate performance-oriented design choice.

- **Thread Safety:** **This class is not thread-safe.** Due to its public mutable fields, concurrent access from multiple threads without external synchronization will lead to race conditions and unpredictable behavior.

    **WARNING:** Never share a Vector3i instance between threads. If a coordinate must be passed to another thread, create a new instance or use the *clone()* method to provide a safe copy.

## API Surface
The public API is focused on construction, serialization, and value-based operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vector3i(int, int, int) | constructor | O(1) | Creates a new vector with specified coordinates. |
| deserialize(ByteBuf, int) | static Vector3i | O(1) | Constructs a Vector3i by reading 12 bytes from a network buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the vector's 12 bytes of data into a network buffer. |
| clone() | Vector3i | O(1) | Creates and returns a new Vector3i instance with the same coordinate values. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer has sufficient bytes to contain a Vector3i at the given offset. |

## Integration Patterns

### Standard Usage
Vector3i is intended for direct instantiation and manipulation within a single-threaded context, such as the main game loop or a network event handler.

```java
// Example: Calculating an adjacent block position
Vector3i playerPosition = new Vector3i(10, 64, -30);
Vector3i blockInFront = playerPosition.clone();
blockInFront.z -= 1; // Directly manipulate public field

// Example: Reading from a network packet
// (Simplified - buf and offset would be provided by the network layer)
Vector3i receivedPosition = Vector3i.deserialize(networkBuffer, offset);
world.updateBlockAt(receivedPosition);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not store a Vector3i instance in a shared location (e.g., a static field) and modify it from multiple systems. This will cause severe and hard-to-debug state corruption.

    ```java
    // ANTI-PATTERN: A shared global object is modified by different systems
    public static final Vector3i DANGER_GLOBAL_POSITION = new Vector3i(0,0,0);

    // Thread 1:
    DANGER_GLOBAL_POSITION.x = 100; // Race condition!

    // Thread 2:
    DANGER_GLOBAL_POSITION.x = -50; // Race condition!
    ```

- **Ignoring Network Byte Order:** The *serialize* and *deserialize* methods use little-endian byte order (`getIntLE`, `writeIntLE`). Do not attempt to interface this class with systems expecting big-endian data without proper byte swapping.

## Data Pipeline
As a core protocol object, Vector3i is a primary payload type. Its data flows directly from the network socket into the game's logic systems.

> Flow:
> Raw TCP Packet -> Netty ByteBuf -> **Vector3i.deserialize** -> Game Event (e.g., BlockUpdate) -> World System -> State Change

