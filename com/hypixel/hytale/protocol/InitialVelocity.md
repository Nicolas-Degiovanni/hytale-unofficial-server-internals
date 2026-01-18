---
description: Architectural reference for InitialVelocity
---

# InitialVelocity

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class InitialVelocity {
```

## Architecture & Concepts
The InitialVelocity class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It is not a service or a manager, but rather a fundamental data structure representing the initial movement vector of a game entity, such as a projectile or a newly spawned character.

Its architecture is optimized for a fixed-size memory layout, occupying exactly 25 bytes within a network buffer. This rigid structure is critical for the protocol's deserialization performance, as it allows parsers to read or skip the entire data block with a single, predictable offset calculation, avoiding conditional logic and variable-length reads.

The class manages three optional fields of type Rangef: yaw, pitch, and speed. Nullability is handled at the protocol level via a single-byte bitmask, an efficient technique that avoids the overhead of larger null indicators and maintains the fixed-size contract.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1. By the static `deserialize` method when the network layer is parsing an incoming packet from a Netty ByteBuf.
    2. By game logic constructing an outgoing network packet that requires velocity information.
- **Scope:** The object's lifetime is extremely short and bound to the scope of a single network packet's processing cycle. It is a transient, short-lived object.
- **Destruction:** The object is eligible for garbage collection as soon as the parent network packet is processed and discarded. There is no manual cleanup or destruction logic.

## Internal State & Concurrency
- **State:** The internal state is mutable. The public fields yaw, pitch, and speed can be modified after instantiation. However, the class is intended to be treated as immutable after being deserialized or before being serialized.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data container with no internal locking or synchronization. It is designed to be created, populated, and read within a single thread, typically a Netty I/O worker or the main game update thread. Unsynchronized access from multiple threads will lead to data corruption and unpredictable behavior.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | InitialVelocity | O(1) | **Static Factory.** Reads 25 bytes from the buffer and constructs a new InitialVelocity object. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 25-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 25. |
| validateStructure(buf, offset) | ValidationResult | O(1) | **Static.** Performs a bounds check to ensure at least 25 bytes are readable from the offset. |
| clone() | InitialVelocity | O(1) | Creates a deep copy of the object and its contained Rangef fields. |

## Integration Patterns

### Standard Usage
InitialVelocity is almost never used directly. It is embedded within larger network packet objects. The parent packet's serialization and deserialization logic delegates to the static methods of this class.

```java
// Example of a parent packet deserializing this object
public void deserialize(ByteBuf buf) {
    // ... deserialize other packet fields
    int currentOffset = ...;
    this.velocity = InitialVelocity.deserialize(buf, currentOffset);
    // ... continue deserializing
}

// Example of game logic creating an instance for an outgoing packet
Rangef speed = new Rangef(10.0f, 15.0f);
Rangef pitch = new Rangef(-1.0f, 1.0f);
InitialVelocity velocity = new InitialVelocity(null, pitch, speed);
SomeSpawnPacket packet = new SomeSpawnPacket(entityId, velocity);
networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Modification:** Do not modify an InitialVelocity object after it has been serialized to a buffer. The serialized data will not reflect the changes.
- **Cross-Thread Sharing:** Never share an instance of InitialVelocity between threads without explicit, external synchronization. This will cause severe race conditions.
- **Variable Size Assumption:** Do not write code that assumes the size of this object can change. The entire design is predicated on its fixed 25-byte size. Rely on `FIXED_BLOCK_SIZE` for any offset calculations.

## Data Pipeline
The class serves as a data marshalling component at the edge of the network layer. It translates between a raw byte representation and a structured Java object.

> **Flow (Incoming Packet):**
> Raw TCP Socket -> Netty ByteBuf -> Protocol Frame Decoder -> **InitialVelocity.deserialize** -> Parent Packet Object -> Game Logic Consumer

