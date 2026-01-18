---
description: Architectural reference for Size
---

# Size

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Size {
```

## Architecture & Concepts
The Size class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a general-purpose dimensional class for game logic; its design is exclusively optimized for network serialization and deserialization.

Its primary architectural role is to provide a standardized, fixed-width binary representation for two-dimensional integer values (width and height). By enforcing a constant 8-byte structure (4 bytes for width, 4 for height), it eliminates the overhead and complexity of variable-length encoding for this common data type. This design choice prioritizes raw performance and predictable buffer offsets, which are critical for the high-throughput demands of the game's networking engine.

This class operates directly on Netty's ByteBuf, tightly coupling it to the underlying network transport layer. All multi-byte integer operations use little-endian byte order, a critical protocol-level convention.

## Lifecycle & Ownership
- **Creation:** Instances are created ephemerally. They are typically instantiated by higher-level protocol message deserializers when parsing an incoming data stream or by game systems preparing an outgoing packet.
- **Scope:** The lifetime of a Size object is extremely short, usually confined to the scope of a single method responsible for packet processing. It is created, its data is used, and it is then immediately eligible for garbage collection.
- **Destruction:** Ownership is never transferred. The object is managed entirely by the Java Garbage Collector and requires no manual cleanup.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The public fields *width* and *height* can be modified directly after instantiation. The class holds no other state and performs no caching.

- **Thread Safety:** This class is **not thread-safe**. Its mutable public fields create a high risk of data races if a single instance is shared and modified concurrently by multiple threads.

    **WARNING:** Never share a Size instance across threads without explicit external synchronization. The intended pattern is to create a new instance per thread or per operation.

## API Surface
The public API is designed for low-level network buffer manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | Size | O(1) | **Static Factory.** Reads 8 bytes from the buffer at the given offset and constructs a new Size object. Does not advance the buffer's reader index. |
| serialize(ByteBuf) | void | O(1) | Writes the width and height (8 bytes total) into the provided buffer using little-endian byte order. Advances the buffer's writer index. |
| computeSize() | int | O(1) | Returns the constant binary size of the structure, which is always 8. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a bounds check to ensure at least 8 bytes are readable from the specified offset. Returns OK or an error result. |

## Integration Patterns

### Standard Usage
The class is almost exclusively used within the serialization and deserialization logic of a larger network message. It is an implementation detail of the protocol, not a component to be used directly by high-level game logic.

```java
// Deserializing a Size from a parent message's buffer
public void deserializeMessage(ByteBuf messageBuffer) {
    // ... read other message fields ...
    int currentOffset = ...;
    Size iconDimensions = Size.deserialize(messageBuffer, currentOffset);
    // ... use iconDimensions ...
}

// Serializing a Size into an outgoing buffer
public void serializeMessage(ByteBuf outBuffer) {
    Size playerAvatarSize = new Size(64, 64);
    // ... write other message fields ...
    playerAvatarSize.serialize(outBuffer);
    // ... continue writing ...
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a single Size instance for multiple network messages by modifying its fields. This is error-prone and offers no performance benefit. Always create a new instance for clarity and safety.
- **Cross-Thread Sharing:** Do not pass a Size object to another thread for processing. Its mutability makes this pattern unsafe without locks, which would defeat the performance-oriented design of the class.
- **Game Logic Calculations:** Avoid using this class for general-purpose 2D vector math or geometry within the game engine. It lacks the necessary methods and is designed specifically for I/O. Use a dedicated vector or dimension class from the game's math library instead.

## Data Pipeline
The Size class is a component in the network data (de)serialization pipeline. It acts as a low-level parser and formatter for a specific 8-byte segment of a raw byte stream.

> **Incoming Data Flow:**
> Raw TCP Packet -> Netty ByteBuf -> Protocol Message Deserializer -> **Size.deserialize(buf, offset)** -> Populated Message Object -> Game Logic
>
> **Outgoing Data Flow:**
> Game Logic -> New Message Object (with a Size field) -> Protocol Message Serializer -> **size.serialize(buf)** -> Netty ByteBuf -> Raw TCP Packet

