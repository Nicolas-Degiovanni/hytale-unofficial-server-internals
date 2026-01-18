---
description: Architectural reference for BlockIdMatcher
---

# BlockIdMatcher

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO) / Transient

## Definition
```java
// Signature
public class BlockIdMatcher {
```

## Architecture & Concepts
The BlockIdMatcher is a fundamental data structure within the Hytale network protocol layer. It is not a service or manager, but a pure data container designed to represent a query or filter for a specific block type. Its primary function is to define the precise binary wire format for matching blocks based on their string identifier, an optional block state, and a numerical tag index.

The architecture of this class is heavily optimized for network performance and bandwidth efficiency. It employs a custom binary layout consisting of two main parts:
1.  A **fixed-size header** of 13 bytes. This header contains a bitmask for nullable fields and offsets to the variable data section.
2.  A **variable-size data block** that stores the actual string content for the *id* and *state* fields.

This fixed-plus-variable layout is a common high-performance pattern in game networking, allowing for extremely fast parsing and size calculation. The use of a `nullBits` byte as a bitmask is a classic optimization to avoid transmitting empty strings, further reducing payload size. The class itself, particularly its static serialization and deserialization methods, serves as the canonical implementation of this part of the network protocol.

## Lifecycle & Ownership
- **Creation:** A BlockIdMatcher instance is created under two primary circumstances:
    1.  **Deserialization:** The network protocol layer instantiates the object via the static `deserialize` method when processing an incoming network packet from a Netty ByteBuf.
    2.  **Manual Instantiation:** Game logic systems (e.g., world editing tools, build systems) create instances via its constructors to populate a packet that will be sent over the network.

- **Scope:** The object is highly **transient**. Its lifetime is typically scoped to the processing of a single network packet. It is created, its data is consumed by a handler, and it is then discarded.

- **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. Once all references to the instance are released, it becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The BlockIdMatcher is a **fully mutable** object. Its public fields, *id*, *state*, and *tagIndex*, can be directly modified at any time after instantiation. This design choice facilitates the easy construction of the object before it is passed to the serialization pipeline.

- **Thread Safety:** This class is **not thread-safe**. Direct access to its public fields makes it inherently unsafe for concurrent use without external synchronization.

    **Warning:** It is fundamentally unsafe to modify a BlockIdMatcher instance on one thread while another thread is serializing it or reading its fields. The intended and safe usage pattern confines a single instance to a single thread, most often a Netty I/O worker thread.

## API Surface
The public API is dominated by static utility methods that operate on Netty byte buffers, forming a complete codec for this data type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BlockIdMatcher | O(N) | Constructs a new BlockIdMatcher by reading from a buffer at a given offset. N is the combined length of the string fields. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided buffer according to the defined binary format. N is the combined length of the string fields. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the total byte size of a serialized BlockIdMatcher within a buffer without performing a full deserialization. Essential for advancing a buffer's read pointer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a series of checks on a buffer to verify if the data at the offset represents a structurally valid BlockIdMatcher. Does not deserialize the object. |
| computeSize() | int | O(N) | Calculates the number of bytes this instance would occupy if it were serialized. N is the combined length of the string fields. |

## Integration Patterns

### Standard Usage
The class is typically used within a network packet handler to read or write data from a stream. The static methods are the primary entry points for I/O operations.

```java
// Example: Deserializing from a packet buffer in a handler
public void handlePacket(ByteBuf packetBuffer) {
    // Validate before attempting to read
    ValidationResult result = BlockIdMatcher.validateStructure(packetBuffer, packetBuffer.readerIndex());
    if (!result.isOk()) {
        throw new ProtocolException("Invalid BlockIdMatcher: " + result.getReason());
    }

    BlockIdMatcher matcher = BlockIdMatcher.deserialize(packetBuffer, packetBuffer.readerIndex());
    int bytesConsumed = BlockIdMatcher.computeBytesConsumed(packetBuffer, packetBuffer.readerIndex());

    // Advance the buffer's reader index past this object
    packetBuffer.skipBytes(bytesConsumed);

    // Now, use the fully-formed matcher object
    processBlockQuery(matcher);
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not share a BlockIdMatcher instance between threads. Modifying its public fields from one thread while another thread calls `serialize` will lead to corrupted network data and unpredictable behavior.
- **Ignoring Validation:** Never call `deserialize` on an untrusted or unvalidated ByteBuf. Always call `validateStructure` first to prevent `ProtocolException` or other buffer-related errors that could crash the network pipeline.
- **Reusing Instances for Serialization:** Do not reuse a single BlockIdMatcher instance to serialize different data in a loop without re-setting all fields. Because fields are nullable, failing to null out a field from a previous iteration can lead to it being unintentionally included in the next serialization.

## Data Pipeline
The BlockIdMatcher acts as a bridge between in-memory Java objects and the raw byte stream of the network protocol.

**Outbound (Serialization) Flow:**
> Game Logic -> Creates and populates **BlockIdMatcher** instance -> `serialize()` method -> Netty ByteBuf -> Network Socket

**Inbound (Deserialization) Flow:**
> Network Socket -> Netty ByteBuf -> `deserialize()` static method -> Creates **BlockIdMatcher** instance -> Packet Handler Logic

