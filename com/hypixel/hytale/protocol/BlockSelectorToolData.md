---
description: Architectural reference for BlockSelectorToolData
---

# BlockSelectorToolData

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BlockSelectorToolData {
```

## Architecture & Concepts
The BlockSelectorToolData class is a specialized Data Transfer Object (DTO) designed for network serialization. It represents a fixed-size, 4-byte data structure within the Hytale network protocol. Its sole purpose is to encapsulate a single game parameter, `durabilityLossOnUse`, for a tool that selects blocks.

This class acts as a fundamental building block for higher-level network packets. It provides a strongly-typed, structured view over a raw segment of a Netty ByteBuf. By standardizing the serialization and deserialization logic, it ensures that data sent from the server is interpreted identically by the client, and vice-versa. The design prioritizes performance and low garbage collection overhead through static factory methods and predictable, fixed-size memory operations.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the network protocol layer. The static `deserialize` method is invoked by a parent packet decoder when it needs to parse this specific data structure from an incoming network buffer. On the sending side, game logic may instantiate it directly before passing it to a serializer.
- **Scope:** Extremely transient. An instance of BlockSelectorToolData is intended to live only for the duration of a single network packet's processing cycle. It is created, its data is consumed by game logic, and it is immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no explicit cleanup methods. Holding long-term references to these objects is considered a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The internal state consists of a single public float, `durabilityLossOnUse`. The state is mutable, which is a design choice for performance to avoid creating new objects for minor modifications before serialization. However, in its primary deserialized use case, it should be treated as an immutable value object.
- **Thread Safety:** This class is **not thread-safe**. The public field can be read or written by any thread, and no synchronization mechanisms are in place. It is designed to be confined to a single thread, typically a Netty I/O worker thread, for the entire duration of its lifecycle.

**WARNING:** Sharing instances of BlockSelectorToolData across threads without external locking will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or pass them between thread contexts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BlockSelectorToolData | O(1) | Constructs a new instance by reading 4 bytes from the buffer at the given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal state (4 bytes) into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 4. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer contains enough readable bytes (at least 4) from the offset. |
| clone() | BlockSelectorToolData | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
The class is intended to be used by higher-level protocol handlers to read or write structured data to a network buffer.

**Deserialization (Receiving Data):**
```java
// A packet handler receives a buffer from the network pipeline
void handlePacket(ByteBuf buffer) {
    // Assume offset is calculated by the parent packet structure
    int dataOffset = 12;

    ValidationResult result = BlockSelectorToolData.validateStructure(buffer, dataOffset);
    if (!result.isOk()) {
        // Handle error, possibly disconnect the client
        throw new ProtocolException(result.getErrorMessage());
    }

    BlockSelectorToolData toolData = BlockSelectorToolData.deserialize(buffer, dataOffset);
    
    // Use the data in game logic
    applyToolProperties(toolData.durabilityLossOnUse);
    
    // The toolData object is now discarded
}
```

**Serialization (Sending Data):**
```java
// Game logic prepares data to be sent
BlockSelectorToolData toolData = new BlockSelectorToolData(0.5f);

// A packet serializer writes it to an outgoing buffer
ByteBuf outgoingBuffer = ...;
toolData.serialize(outgoingBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Retaining Instances:** Do not cache or store instances of BlockSelectorToolData in long-lived objects. They are cheap to create and should be discarded after use to avoid unnecessary memory pressure.
- **Ignoring Validation:** Failure to call `validateStructure` before `deserialize` on untrusted input can result in an IndexOutOfBoundsException, crashing the network thread.
- **Concurrent Access:** As detailed in the concurrency section, accessing a single instance from multiple threads is unsafe and must be avoided.

## Data Pipeline
BlockSelectorToolData serves as a deserialization endpoint in the network data pipeline. It transforms a raw byte stream into a structured, usable Java object.

> Flow (Inbound):
> Raw TCP Stream -> Netty ByteBuf -> Parent Packet Decoder -> **BlockSelectorToolData.deserialize()** -> Game Logic Consumer

> Flow (Outbound):
> Game Logic Producer -> **new BlockSelectorToolData()** -> Parent Packet Encoder -> **BlockSelectorToolData.serialize()** -> Netty ByteBuf -> Raw TCP Stream

