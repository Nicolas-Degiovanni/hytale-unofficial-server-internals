---
description: Architectural reference for RailPoint
---

# RailPoint

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (Transient)

## Definition
```java
// Signature
public class RailPoint {
```

## Architecture & Concepts
The RailPoint class is a low-level, high-performance data structure used within the Hytale network protocol layer. It is not a high-level game entity but a fundamental building block for serializing and deserializing spatial data. Its primary purpose is to represent a single point on a path or spline, defined by a position vector and a normal vector.

The core architectural principle of RailPoint is its **fixed-size binary layout**. The entire object is guaranteed to occupy exactly 25 bytes on the wire. This design choice is critical for network performance, as it eliminates the need for dynamic size calculations, reduces memory allocation overhead, and simplifies buffer management on both the client and server.

This class acts as a direct bridge between raw network byte streams, managed by Netty, and the in-memory representation of game data. Its serialization and deserialization logic is tightly coupled to the Netty ByteBuf, indicating its position at the lowest level of the protocol object model.

## Lifecycle & Ownership
- **Creation:** RailPoint instances are created under two primary circumstances:
    1.  By a higher-level packet deserializer which invokes the static `deserialize` method when parsing an incoming network message.
    2.  By game logic when constructing a data packet to be sent over the network.
    Instances are never managed by a dependency injection framework or service locator.

- **Scope:** The lifecycle of a RailPoint is extremely short. It is a transient object, typically scoped to the processing of a single network packet or a single game tick. It is designed to be created, used, and then immediately discarded.

- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which usually occurs upon completion of the network event or game logic block that created it. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency
- **State:** Mutable. The public fields `point` and `normal` can be modified after instantiation. However, in practice, instances are treated as immutable values after being deserialized or before being serialized.

- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields are not protected by any synchronization mechanisms.

    **WARNING:** Sharing a RailPoint instance across multiple threads without explicit external locking will lead to race conditions and unpredictable behavior. Instances should be confined to the thread that created them, such as a Netty event loop thread or the main game thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static RailPoint | O(1) | Constructs a RailPoint by reading a fixed 25-byte block from a buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a fixed 25-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 25. |
| clone() | RailPoint | O(1) | Performs a deep copy of the object and its contained Vector3f members. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer contains enough readable bytes for a valid RailPoint structure. |

## Integration Patterns

### Standard Usage
A RailPoint is almost never used in isolation. It is typically processed as part of a larger data packet's serialization or deserialization routine. The static `deserialize` method is the primary entry point for inbound data.

```java
// Example: Reading a RailPoint from a network buffer within a packet decoder

// Assume 'packetBuffer' is a Netty ByteBuf received from the network
// and 'readIndex' is the current position within that buffer.

ValidationResult result = RailPoint.validateStructure(packetBuffer, readIndex);
if (!result.isOk()) {
    throw new ProtocolException("Invalid buffer for RailPoint: " + result.getMessage());
}

RailPoint pointData = RailPoint.deserialize(packetBuffer, readIndex);
int bytesConsumed = RailPoint.computeBytesConsumed(packetBuffer, readIndex);
readIndex += bytesConsumed;

// The 'pointData' object is now ready for use by game logic.
```

### Anti-Patterns (Do NOT do this)
- **Holding Long-Lived References:** Do not store RailPoint instances in caches, game state components, or other long-lived objects. They are meant for transient data transfer, not for representing persistent state. Convert them to a higher-level game-specific type if persistence is needed.

- **Cross-Thread Sharing:** Never pass a RailPoint instance from a network thread to a game logic thread (or vice-versa) without deep-copying it or ensuring a proper memory-safe handoff. The lack of thread safety makes direct sharing extremely dangerous.

- **Ignoring Fixed-Size Contract:** Do not attempt to write custom serialization logic that results in a variable size. The entire protocol relies on the assumption that a RailPoint is exactly 25 bytes. Modifying this contract will cause widespread deserialization failures.

## Data Pipeline
The RailPoint class is a fundamental component in the network data pipeline. It represents a single, atomic piece of structured data translated from a raw byte stream.

> **Inbound Flow:**
> Raw TCP Stream -> Netty ByteBuf -> Packet Deserializer -> **RailPoint.deserialize()** -> Game Logic (e.g., Pathfinding System)
>
> **Outbound Flow:**
> Game Logic -> **new RailPoint()** -> Packet Serializer -> **RailPoint.serialize()** -> Netty ByteBuf -> Raw TCP Stream

