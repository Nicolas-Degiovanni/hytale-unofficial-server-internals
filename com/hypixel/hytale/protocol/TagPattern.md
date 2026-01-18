---
description: Architectural reference for TagPattern
---

# TagPattern

**Package:** com.hypixel.hytale.protocol
**Type:** Data Model

## Definition
```java
// Signature
public class TagPattern {
```

## Architecture & Concepts

The TagPattern class is a self-contained data model that represents a recursive, tree-like logical structure for network communication. It is a fundamental component of the Hytale protocol layer, designed to encode complex queries or filters—such as those used for entity selection—into a highly optimized binary format.

Architecturally, this class serves as both a data structure and its own serializer/deserializer. It does not depend on external serialization frameworks, instead implementing a custom, performance-critical binary protocol directly on top of Netty's ByteBuf.

The binary format is a key concept. Each TagPattern object is encoded as a block of data containing two distinct regions:
1.  **Fixed-Size Header (14 bytes):** This section contains a null-bit field, the pattern type, a tag index, and two integer offsets.
2.  **Variable-Size Data Region:** This section immediately follows the header and contains the serialized data for child patterns (the operands array and the not pattern). The offsets in the fixed-size header point to the start of these child structures within this variable region.

This design allows for efficient parsing by reading a fixed-size block to determine the presence and location of subsequent variable-length data, avoiding the need to scan the entire byte stream. The recursive nature of the class, where the operands and not fields are themselves TagPattern instances, allows for the construction of arbitrarily complex logical trees (e.g., `(tagA AND tagB) OR NOT (tagC)`).

## Lifecycle & Ownership

-   **Creation:** A TagPattern instance is created in one of two ways:
    1.  **Deserialization:** The static method TagPattern.deserialize is called by a higher-level protocol decoder when a network packet containing a TagPattern is received. This is the primary creation path for inbound data.
    2.  **Direct Instantiation:** Game logic creates a new TagPattern instance using its constructors to build a query tree that will be sent over the network. This is the primary creation path for outbound data.

-   **Scope:** Instances are transient and short-lived. They exist only for the duration of a single network operation—either to be serialized into an outgoing buffer or to be used by game logic after being deserialized from an incoming buffer. They are not intended to be stored in long-term state.

-   **Destruction:** The object is managed by the Java Garbage Collector. There are no manual resource management or cleanup methods. Once all references to an instance are dropped, it becomes eligible for collection.

## Internal State & Concurrency

-   **State:** The TagPattern class is **mutable**. Its public fields can be modified after instantiation. This facilitates the programmatic construction of complex pattern trees before serialization.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms.

    **Warning:** Accessing or modifying a TagPattern instance from multiple threads concurrently will lead to data corruption and unpredictable behavior. All operations on a given instance must be confined to a single thread, typically a Netty I/O thread or a main game logic thread.

## API Surface

The public API is dominated by static utility methods for serialization and validation, reflecting its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static TagPattern | O(N) | Constructs a TagPattern tree from a binary representation in a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Serializes the current TagPattern instance and its entire child tree into the provided ByteBuf. |
| computeSize() | int | O(N) | Recursively calculates the exact number of bytes required to serialize the entire pattern tree. Useful for pre-allocating buffers. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the binary data in a ByteBuf to ensure it represents a valid TagPattern structure without full deserialization. |
| clone() | TagPattern | O(N) | Creates a deep copy of the TagPattern instance and all its children. |

*N represents the total number of nodes in the TagPattern tree.*

## Integration Patterns

### Standard Usage

The primary use case involves either building a pattern for an outbound packet or parsing one from an inbound packet. The class is designed to be handled by the network protocol layer.

```java
// Example: Deserializing a TagPattern from a network buffer
// This code would typically reside within a packet handler.

ByteBuf incomingBuffer = ... // Received from Netty pipeline

try {
    // Assuming the pattern starts at the current reader index
    TagPattern pattern = TagPattern.deserialize(incomingBuffer, incomingBuffer.readerIndex());

    // The buffer's reader index is NOT advanced by deserialize, so we must advance it manually
    int bytesConsumed = TagPattern.computeBytesConsumed(incomingBuffer, incomingBuffer.readerIndex());
    incomingBuffer.skipBytes(bytesConsumed);

    // Now, use the fully-formed pattern in game logic
    // e.g., world.findEntities(pattern);

} catch (ProtocolException e) {
    // Handle malformed packet data
    // e.g., log the error and disconnect the client
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not share a TagPattern instance between threads. The object is not designed for concurrent access and lacks any synchronization.

-   **Manual Buffer Manipulation:** Do not attempt to read or write the TagPattern binary format manually. The format, with its null-bit fields and relative offsets, is complex and implementation-specific. Always use the provided serialize and deserialize methods.

-   **Ignoring Buffer Position:** The deserialize and computeBytesConsumed methods operate on a given offset and do not modify the ByteBuf's reader or writer index. Failing to manually advance the reader index after a successful deserialization will cause subsequent reads from the buffer to fail or process the same data again.

## Data Pipeline

The TagPattern class is a critical link in the data flow between the raw network stream and structured game logic.

**Inbound Flow (Receiving Data):**
> Flow:
> Netty ByteBuf -> Protocol Packet Decoder -> **TagPattern.deserialize** -> TagPattern Instance -> Game Logic (e.g., Query System)

**Outbound Flow (Sending Data):**
> Flow:
> Game Logic -> New TagPattern Instance -> **instance.serialize(ByteBuf)** -> Protocol Packet Encoder -> Netty ByteBuf

