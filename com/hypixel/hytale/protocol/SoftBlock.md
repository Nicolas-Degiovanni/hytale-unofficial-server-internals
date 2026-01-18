---
description: Architectural reference for SoftBlock
---

# SoftBlock

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class SoftBlock {
```

## Architecture & Concepts
The SoftBlock class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It represents the state of a single block-like entity within the Hytale protocol. Its primary role is to serve as a structured, serializable data contract between the client and server, ensuring that block data is transmitted with minimal overhead.

The architecture of its binary format is heavily optimized for both size and processing speed:

*   **Fixed-Size Header:** The serialized structure begins with a 10-byte fixed-size header. This predictable size allows for efficient parsing and validation.
*   **Nullability Bitmask:** The first byte of the header is a bitmask (nullBits) that indicates which of the nullable, variable-length fields (itemId, dropListId) are present in the payload. This avoids the need to write placeholder values for null fields, saving network bandwidth.
*   **Offset-Based Pointers:** The header contains 4-byte integer offsets that point to the location of variable-length data within the buffer. This "pointer" system decouples the fixed-size header from the variable-size data, allowing parsers to quickly locate specific fields without reading through all preceding data.
*   **Variable-Length Encoding:** String lengths and the strings themselves are written using variable-length integer (VarInt) encoding and UTF-8, further minimizing the byte footprint for common values.

This design pattern is critical for systems like game engines where millions of such objects may be serialized and deserialized per second.

## Lifecycle & Ownership
- **Creation:** A SoftBlock instance is created under two primary circumstances:
    1.  **Inbound:** The static factory method SoftBlock.deserialize is invoked by a higher-level protocol decoder when parsing an incoming network packet from a ByteBuf.
    2.  **Outbound:** Game logic instantiates a new SoftBlock via its constructor (e.g., new SoftBlock(...)) to prepare data for transmission.
- **Scope:** The object is transient and has a very short lifetime. It is typically scoped to the processing of a single network packet. Once the packet has been handled or sent, the SoftBlock instance is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual resource management or cleanup methods.

## Internal State & Concurrency
- **State:** The class is **mutable**. Its public fields can be modified directly after instantiation. This design prioritizes performance by avoiding the overhead of creating new objects for minor state changes, but it introduces risks if not handled correctly.
- **Thread Safety:** SoftBlock is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All operations on a SoftBlock instance must be confined to a single thread, typically a Netty I/O worker or the main game logic thread.

**WARNING:** Sharing SoftBlock instances across threads without external synchronization will lead to race conditions, data corruption, and unpredictable behavior.

## API Surface
The public API is divided into static methods for operating on raw ByteBufs and instance methods for operating on an object's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static SoftBlock | O(N) | Constructs a new SoftBlock by reading from a ByteBuf at a given offset. N is the size of the variable data. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the given ByteBuf according to the defined binary format. N is the size of the variable data. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the total size of a serialized SoftBlock within a buffer without performing a full deserialization. Extremely fast. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. N is the size of the variable data. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a structural integrity check on a potential SoftBlock within a buffer. Does not validate content, only offsets and lengths. |

## Integration Patterns

### Standard Usage
A SoftBlock is almost never used in isolation. It is created and processed by higher-level packet handlers that manage the network buffer.

```java
// Example: Reading a SoftBlock from a larger packet's payload
public void read(ByteBuf packetBuffer) {
    // ... read other packet fields ...
    int blockDataOffset = packetBuffer.readerIndex();
    SoftBlock.validateStructure(packetBuffer, blockDataOffset);
    this.block = SoftBlock.deserialize(packetBuffer, blockDataOffset);
    // Advance the buffer's reader index
    packetBuffer.readerIndex(blockDataOffset + SoftBlock.computeBytesConsumed(packetBuffer, blockDataOffset));
    // ... continue reading packet ...
}

// Example: Writing a SoftBlock to a packet's payload
public void write(ByteBuf packetBuffer) {
    // ... write other packet fields ...
    SoftBlock blockToWrite = new SoftBlock("hytale:stone", null, false);
    blockToWrite.serialize(packetBuffer);
    // ... continue writing packet ...
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold onto a SoftBlock instance after its parent packet has been processed. Its mutable nature makes it unsafe for caching or re-use without explicit deep cloning.
- **Multi-threaded Access:** Never pass a SoftBlock instance to another thread for modification. If data must be shared, either create a deep copy (using the copy constructor) or extract the data into an immutable, thread-safe structure.
- **Manual Construction for Deserialization:** Do not use `new SoftBlock()` and then manually populate its fields from a ByteBuf. This bypasses the complex logic in the `deserialize` method and will almost certainly fail.

## Data Pipeline
The SoftBlock acts as a translation layer between the raw byte stream of the network and the structured object model of the game logic.

> **Inbound Flow:**
> Netty ByteBuf -> Protocol Packet Decoder -> **SoftBlock.deserialize()** -> SoftBlock Instance -> Game Logic/Event Bus
>
> **Outbound Flow:**
> Game Logic -> new SoftBlock(...) -> Protocol Packet Encoder -> **instance.serialize()** -> Netty ByteBuf

