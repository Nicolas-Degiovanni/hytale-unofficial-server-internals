---
description: Architectural reference for BlockGroup
---

# BlockGroup

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class BlockGroup {
```

## Architecture & Concepts
The BlockGroup class is a low-level Data Transfer Object (DTO) designed for network serialization. It is not a service or manager, but rather a passive data structure that represents a collection of block names within the Hytale network protocol.

Its primary role is to provide a standardized, efficient, and safe mechanism for encoding and decoding an array of strings to and from a Netty ByteBuf. The serialization format is highly optimized for network transmission, utilizing a leading bitmask to handle null fields and VarInt encoding for array and string lengths to minimize byte footprint.

This class is a fundamental building block for more complex network packets. It encapsulates the precise binary layout of its data, abstracting the complexities of byte manipulation, bounds checking, and validation away from higher-level protocol handlers.

## Lifecycle & Ownership
- **Creation:** BlockGroup instances are created under two primary scenarios:
    1.  **Deserialization:** The static factory method **deserialize** is called by a network protocol handler when reading an incoming data stream. This creates a new BlockGroup instance populated with data from the network buffer.
    2.  **Manual Instantiation:** A developer creates an instance using **new BlockGroup(names)** when preparing data to be sent over the network. This instance is then passed to a serializer.

- **Scope:** The object's lifetime is ephemeral. It is designed to be short-lived, typically existing only for the duration of processing a single network packet. It is created, used immediately for reading or writing, and then becomes eligible for garbage collection.

- **Destruction:** The Java Garbage Collector is responsible for deallocating the object once it is no longer referenced. There are no manual cleanup or close methods.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a single nullable array of strings, **names**. The object does not cache any data beyond this field.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization. It is designed to be created, populated, and used within a single thread, such as a Netty I/O worker thread.

    **WARNING:** Sharing a BlockGroup instance across multiple threads without external synchronization will lead to race conditions and undefined behavior. Do not store instances in shared state.

## API Surface
The public API is divided into static methods for reading from a buffer and instance methods for writing to one.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BlockGroup | O(N) | Constructs a new BlockGroup by reading from a ByteBuf. Throws ProtocolException on malformed data or buffer underflow. N is the total number of bytes in the string data. |
| serialize(buf) | void | O(N) | Encodes the instance's state into the provided ByteBuf. Throws ProtocolException if data constraints are violated (e.g., array too long). |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy when serialized. Does not modify any buffers. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Scans a ByteBuf to determine the size of a serialized BlockGroup without performing a full deserialization. Crucial for advancing buffer read pointers correctly. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a lightweight structural validation on the data in a buffer. Used for pre-flight checks to prevent deserialization errors. |
| clone() | BlockGroup | O(M) | Creates a shallow copy of the BlockGroup, where M is the number of elements in the names array. |

## Integration Patterns

### Standard Usage
BlockGroup is typically used by higher-level packet objects or protocol codecs to handle a specific field within a larger data structure.

**Deserialization (Reading from the network):**
```java
// A network handler receives a ByteBuf.
// After reading other packet data, it deserializes a BlockGroup.
int currentOffset = ...; // The starting position in the buffer
BlockGroup group = BlockGroup.deserialize(buf, currentOffset);

// Advance the buffer's reader index
int bytesRead = BlockGroup.computeBytesConsumed(buf, currentOffset);
buf.readerIndex(buf.readerIndex() + bytesRead);

// Use the deserialized object
processBlockData(group.names);
```

**Serialization (Writing to the network):**
```java
// Game logic prepares data to be sent.
String[] blockNames = new String[]{"stone", "dirt", "grass"};
BlockGroup group = new BlockGroup(blockNames);

// A network handler writes it to an outgoing ByteBuf.
group.serialize(outputBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not deserialize into an existing BlockGroup instance or re-use a single instance for multiple outgoing packets. The object is cheap to create and should be treated as disposable.

- **Incorrect Buffer Management:** Do not manually advance the buffer's reader index after calling **deserialize**. The method reads from an absolute offset and does not modify the buffer's state. Use **computeBytesConsumed** to determine how far to advance the pointer.

- **Cross-Thread Sharing:** Do not create a BlockGroup on one thread and pass it to another for serialization or modification without proper synchronization. This will lead to memory visibility issues and race conditions.

## Data Pipeline
BlockGroup acts as a data marshalling component at the edge of the network layer.

> **Ingress Flow (Read):**
> Netty Channel -> ByteBuf -> Protocol Codec -> **BlockGroup.deserialize** -> BlockGroup Instance -> Game Logic

> **Egress Flow (Write):**
> Game Logic -> BlockGroup Instance -> **instance.serialize** -> Protocol Codec -> ByteBuf -> Netty Channel

