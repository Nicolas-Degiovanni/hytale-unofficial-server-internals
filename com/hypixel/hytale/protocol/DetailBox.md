---
description: Architectural reference for DetailBox
---

# DetailBox

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class DetailBox {
```

## Architecture & Concepts
The DetailBox class is a fundamental data structure within the Hytale network protocol layer. It is not a service or manager, but rather a Data Transfer Object (DTO) designed for high-performance serialization and deserialization. It represents a composite physical volume, combining a positional offset with a bounding box (Hitbox), likely used for precise collision detection, interaction checks, or entity rendering details.

The core architectural principle of DetailBox is its **fixed-size binary layout**. Every DetailBox instance, regardless of its content, serializes to a consistent 37-byte block. This design choice is critical for performance in the network layer, as it simplifies buffer parsing, eliminates the need for variable-length decoding logic, and allows for predictable memory allocation on both the client and server.

To accommodate nullable fields within this fixed-size constraint, DetailBox employs a bitmask strategy. The first byte of the serialized data acts as a "null bit field", where individual bits indicate the presence or absence of the `offset` and `box` fields. If a field is null, its corresponding space in the byte buffer is padded with zeros, ensuring the total size remains constant.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the network protocol layer via the static `deserialize` factory method when decoding an incoming packet.
    2.  By game logic when constructing an object that will be serialized into an outgoing network packet.
- **Scope:** A DetailBox is a short-lived, transient object. Its lifetime is typically confined to the scope of a single network packet's processing cycle or a single game tick. It is not designed to be stored or persisted long-term.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which is typically after the parent network packet has been fully processed or transmitted. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The internal state of a DetailBox is **fully mutable**. Its public fields, `offset` and `box`, can be directly accessed and modified after instantiation. It acts as a simple data container with no internal logic for state management.

- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game loop. Accessing or modifying a DetailBox instance from multiple threads concurrently will result in data corruption and race conditions. Any multi-threaded access must be protected by external synchronization mechanisms.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static DetailBox | O(1) | **Primary Entry Point.** Deserializes a fixed 37-byte block from a buffer into a new DetailBox instance. |
| serialize(ByteBuf) | void | O(1) | Serializes the instance into the provided buffer, writing exactly 37 bytes. |
| computeSize() | int | O(1) | Returns the constant size (37) of the serialized object. |
| clone() | DetailBox | O(1) | Performs a deep copy of the instance, creating new Vector3f and Hitbox objects if they exist. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer has enough readable bytes (37) to contain a valid DetailBox structure. |

## Integration Patterns

### Standard Usage
The most common use case is deserializing a DetailBox from a network buffer as part of a larger packet structure. The static `deserialize` method is the canonical way to create an instance from raw network data.

```java
// Example: Reading a DetailBox from a network ByteBuf
// Assume 'buffer' is a received ByteBuf and 'offset' is the
// starting position of the DetailBox data within the buffer.

ValidationResult result = DetailBox.validateStructure(buffer, offset);
if (result.isError()) {
    // Handle malformed packet
    throw new CorruptedPacketException(result.getErrorMessage());
}

DetailBox detail = DetailBox.deserialize(buffer, offset);

if (detail.box != null) {
    // Perform game logic using the hitbox data
    processCollision(detail.box);
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not share a DetailBox instance across multiple threads. Its mutable nature makes it inherently unsafe for concurrent access. If data must be passed between threads, send an immutable copy or a deep clone.
- **Assuming Variable Size:** Do not attempt to parse a DetailBox as a variable-length structure. The protocol guarantees a fixed 37-byte layout. Any logic that attempts to read more or fewer bytes based on the null bit field will lead to buffer desynchronization and critical parsing errors.
- **Manual Deserialization:** Avoid reading the fields manually from a buffer. The `deserialize` method correctly handles the null bit field and fixed-width padding, which is complex and error-prone to replicate.

## Data Pipeline
DetailBox acts as a data payload within the network protocol pipeline. It does not process data itself; it *is* the data being processed.

> **Ingress Flow (Receiving Data):**
> Raw TCP Stream -> Netty ByteBuf -> Packet Deserializer -> **DetailBox.deserialize** -> Game Logic (as DetailBox object)

> **Egress Flow (Sending Data):**
> Game Logic (creates DetailBox object) -> Packet Serializer -> **DetailBox.serialize** -> Netty ByteBuf -> Raw TCP Stream

