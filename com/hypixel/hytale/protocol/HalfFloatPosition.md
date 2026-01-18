---
description: Architectural reference for HalfFloatPosition
---

# HalfFloatPosition

**Package:** com.hypixel.hytale.protocol
**Type:** Value Object / Data Structure

## Definition
```java
// Signature
public class HalfFloatPosition {
```

## Architecture & Concepts
The HalfFloatPosition class is a fundamental data structure within the Hytale network protocol layer. It serves as a highly optimized, low-level representation of a 3D coordinate. Its primary design goal is to minimize network bandwidth by encoding each coordinate axis (X, Y, Z) as a 16-bit signed integer (a Java short), which is a common technique for representing half-precision floating-point values in performance-critical network code.

This class is not a general-purpose vector or position utility; it is a specialized serialization contract. Its structure is rigid and directly maps to a fixed 6-byte block in the network byte stream. All serialization and deserialization logic is self-contained, operating directly on Netty ByteBuf instances to maximize performance and reduce intermediate object allocations. It is a building block, intended to be composed within larger network packet objects.

## Lifecycle & Ownership
- **Creation:** HalfFloatPosition instances are ephemeral and created frequently. They are primarily instantiated by protocol decoders via the static *deserialize* factory method when parsing an incoming network packet. Game logic will also create instances directly when preparing data to be sent over the network.
- **Scope:** The lifetime of a HalfFloatPosition object is extremely short. It is typically method-scoped or held as a field within a larger packet object. Its lifecycle is bound to the processing of a single network message or a single game tick update.
- **Destruction:** Instances are managed by the Java Garbage Collector. Due to their short-lived nature, they are prime candidates for fast, generational garbage collection. No manual resource management is required.

## Internal State & Concurrency
- **State:** The state is **mutable** and consists of three public short fields: x, y, and z. This design choice prioritizes performance by allowing direct field access, avoiding the overhead of getter and setter method calls in performance-sensitive code paths like the network or game loop.
- **Thread Safety:** This class is **not thread-safe**. The public, mutable fields make it inherently unsafe for concurrent access.

    **WARNING:** Instances of HalfFloatPosition must not be shared between threads without external synchronization. It is designed to be used exclusively within a single-threaded context, such as a Netty I/O thread or the main game logic thread. Unsynchronized multi-threaded access will lead to race conditions and data corruption.

## API Surface
The public API is minimal, focusing exclusively on serialization, state, and value-based equality.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | HalfFloatPosition | O(1) | **Static Factory.** Reads 6 bytes from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | void | O(1) | Writes the internal x, y, and z state as three little-endian shorts into the buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 6 bytes. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Method.** Performs a pre-flight check to ensure a buffer contains enough readable bytes for a valid object. |
| clone() | HalfFloatPosition | O(1) | Creates a new instance with a copy of the x, y, and z values. |

## Integration Patterns

### Standard Usage
HalfFloatPosition is almost never used in isolation. It is embedded within larger packet structures and handled by the protocol serialization engine. The primary interaction points are the static *deserialize* method for reading and the instance *serialize* method for writing.

```java
// Example: Deserializing a position from a network buffer
// This logic would typically reside inside a larger packet's deserialization code.

ByteBuf networkBuffer = ...;
int currentOffset = ...;

// Pre-validate that enough data exists
ValidationResult result = HalfFloatPosition.validateStructure(networkBuffer, currentOffset);
if (!result.isOk()) {
    throw new DeserializationException(result.getErrorMessage());
}

// Deserialize to create the object
HalfFloatPosition playerPos = HalfFloatPosition.deserialize(networkBuffer, currentOffset);

// The game logic can now use the playerPos object
processPlayerPosition(playerPos);
```

### Anti-Patterns (Do NOT do this)
- **Sharing Instances:** Do not pass a HalfFloatPosition instance to another thread. Its mutable nature makes this extremely dangerous. If position data must be shared, either create a clone or pass the primitive short values.
- **Manual Serialization:** Do not manually write the x, y, and z fields to a buffer using methods like *buf.writeShort()*. This bypasses the contract which guarantees little-endian byte order and could lead to cross-platform incompatibility. Always use the provided *serialize* method.
- **Treating as a Physics Vector:** This is a network data container, not a mathematical vector class. Do not attempt to add vector arithmetic methods to it. For game logic or physics calculations, convert its state into a dedicated vector class from a math library.

## Data Pipeline
HalfFloatPosition acts as a data payload that is serialized and deserialized at the boundaries of the application. It does not process data itself; it *is* the data.

**Inbound (Client/Server Receiving Data):**
> Flow:
> Raw TCP Bytes -> Netty ByteBuf -> Protocol Decoder -> **HalfFloatPosition.deserialize()** -> Game Packet Object -> Game Logic

**Outbound (Client/Server Sending Data):**
> Flow:
> Game Logic -> Game Packet Object with HalfFloatPosition -> Protocol Encoder -> **instance.serialize()** -> Netty ByteBuf -> Raw TCP Bytes

