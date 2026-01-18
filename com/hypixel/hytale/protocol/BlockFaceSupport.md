---
description: Architectural reference for BlockFaceSupport
---

# BlockFaceSupport

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class BlockFaceSupport {
```

## Architecture & Concepts
The BlockFaceSupport class is a specialized Data Transfer Object (DTO) designed for high-performance network serialization and deserialization. It is not a service or manager, but rather a data structure that represents a component of a larger network packet. Its primary role is to encapsulate data related to the surface properties of a game world block.

The architecture of this class is heavily optimized for wire-format efficiency. It employs a custom binary layout rather than relying on a generic serialization framework. This layout consists of three distinct parts:

1.  **Nullable Bit Field:** A single byte at the start of the structure. Each bit corresponds to a nullable field (faceType, filler), indicating whether its data is present in the payload. This avoids wasting space for null values.
2.  **Offset Table:** A fixed-size block containing 32-bit integer offsets for each variable-length field. These offsets are relative to the start of the variable data block, not the start of the object itself. This allows a parser to jump directly to a specific field without reading all preceding data.
3.  **Variable Data Block:** The actual payload for fields like strings and arrays, concatenated together.

This design is critical for the protocol layer, enabling efficient partial-reads and validation without requiring full object deserialization.

## Lifecycle & Ownership
- **Creation:** Instances are created under two circumstances:
    1.  **Inbound:** The static factory method BlockFaceSupport.deserialize is called by a higher-level packet parser when reading a payload from a Netty ByteBuf.
    2.  **Outbound:** Game logic instantiates it directly using new BlockFaceSupport(...) when constructing a packet to be sent over the network.
- **Scope:** An instance of BlockFaceSupport is extremely short-lived. Its scope is confined to the processing of a single network packet. It is created, read from or written to, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or close methods. References should not be held after the parent network packet has been processed.

## Internal State & Concurrency
- **State:** The class holds its state in two public, mutable fields: faceType and filler. This design prioritizes raw performance and ease of access over encapsulation. The state is intended to be populated once (either by the constructor or deserializer) and then read.

- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields and lack of any synchronization primitives make it inherently unsafe for concurrent access. It is designed to be created, manipulated, and read exclusively within the context of a single thread, typically a Netty I/O worker thread.

    **WARNING:** Sharing instances of BlockFaceSupport across threads will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or pass them between thread boundaries without deep cloning.

## API Surface
The public API is divided into static utility methods for buffer manipulation and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BlockFaceSupport | O(N) | Constructs a new BlockFaceSupport object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. Throws ProtocolException if constraints are violated. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size in bytes of a serialized BlockFaceSupport structure within a buffer without performing a full deserialization. Critical for advancing a buffer's read pointer. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a series of fast, non-allocating checks to validate if the data at the given offset appears to be a structurally valid BlockFaceSupport. Does not validate content. |
| computeSize() | int | O(1) | Calculates the byte size this object would occupy if it were to be serialized. Useful for pre-allocating buffers. |

## Integration Patterns

### Standard Usage
BlockFaceSupport is almost never used in isolation. It is a component of a larger packet structure. A parent packet's deserializer is responsible for invoking the static methods.

```java
// Example: Deserializing from a parent packet's buffer
// Assume 'packetBuffer' is a ByteBuf and 'currentOffset' is the
// starting position of the BlockFaceSupport data.

ValidationResult result = BlockFaceSupport.validateStructure(packetBuffer, currentOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid BlockFaceSupport structure: " + result.getMessage());
}

BlockFaceSupport faceData = BlockFaceSupport.deserialize(packetBuffer, currentOffset);
int bytesRead = BlockFaceSupport.computeBytesConsumed(packetBuffer, currentOffset);

// Advance the buffer pointer for the next field
currentOffset += bytesRead;

// ... use faceData ...
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not deserialize into an existing BlockFaceSupport instance or reuse a single instance for multiple outgoing packets. The mutable state makes this pattern highly error-prone. Always create a new instance for each distinct operation.
- **Multi-threaded Access:** Do not access an instance from any thread other than the one that created it. If data must be passed to another thread, perform a deep copy using the clone method or by constructing a new object.
- **Ignoring Offset:** The static methods require a precise offset into the buffer. Passing the buffer's readerIndex or an incorrect offset will result in deserialization errors or data corruption.

## Data Pipeline
BlockFaceSupport acts as a translation point between raw bytes and a structured, in-memory representation.

> **Inbound Flow:**
> Raw Network ByteBuf -> Parent Packet Deserializer -> **BlockFaceSupport.deserialize** -> In-Memory BlockFaceSupport Object -> Game Logic Processing

> **Outbound Flow:**
> Game Logic -> new BlockFaceSupport(...) -> Parent Packet Serializer -> **instance.serialize(ByteBuf)** -> Raw Network ByteBuf

