---
description: Architectural reference for ItemQuantity
---

# ItemQuantity

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ItemQuantity {
```

## Architecture & Concepts
The ItemQuantity class is a low-level Data Transfer Object (DTO) that defines the binary wire format for representing an item and its associated count within the Hytale network protocol. It is not a service or a manager, but rather a fundamental building block for more complex packets, such as those related to inventories, crafting, or world entities.

Its primary architectural role is to serve as a concrete, in-memory representation of a serialized data structure. The class encapsulates the precise logic for converting between this Java object and its byte-level representation in a Netty ByteBuf. This includes handling data layout, nullability, and variable-length fields to ensure protocol consistency between the client and server.

The design employs a static utility pattern for its core functionality. Methods like *deserialize* and *serialize* operate directly on byte buffers, making the class both a data container and the canonical codec for its own data format.

### Lifecycle & Ownership
- **Creation:** An ItemQuantity instance is created under two distinct circumstances:
    1.  **Outbound:** By game logic to prepare data for network transmission. For example, when a player moves an item, the inventory system creates a new ItemQuantity to be included in an outgoing packet.
    2.  **Inbound:** By the protocol deserialization layer. The static *deserialize* method is called by a parent packet's deserialization logic to construct an ItemQuantity from an incoming ByteBuf.
- **Scope:** The object's lifetime is intentionally brief and transactional. It typically exists only for the duration of a single network operation—either to be written into a buffer or to be read from one and processed by game logic. It is a classic transient object.
- **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup as soon as it is no longer referenced, which is usually immediately after the network packet has been processed or sent.

## Internal State & Concurrency
- **State:** The state is mutable and consists of two public fields: *itemId* and *quantity*. The class is a simple data holder and performs no internal caching. Its state directly reflects the data it was constructed with or deserialized from.
- **Thread Safety:** This class is **not thread-safe**. Its public fields can be read and written from any thread without synchronization. This is by design, as protocol objects are expected to be confined to a single network processing thread (e.g., a Netty event loop) to avoid the overhead of synchronization.

**WARNING:** Modifying an ItemQuantity instance from multiple threads will lead to race conditions and unpredictable behavior. All interaction should occur on the thread responsible for processing the network packet.

## API Surface
The primary contract is defined by the static serialization and validation methods, not the instance fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ItemQuantity | O(N) | Constructs an ItemQuantity by reading from a ByteBuf at a given offset. N is the length of the itemId string. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the instance's state into the provided ByteBuf according to the protocol specification. N is the length of the itemId string. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this instance will consume when serialized. Useful for pre-allocating buffers. N is the length of the itemId string. |
| computeBytesConsumed(buf, offset) | int | O(N) | Reads just enough of the buffer to determine the total size of a serialized ItemQuantity without fully deserializing it. N is the length of the itemId string. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | Performs a lightweight check on a buffer to ensure a valid ItemQuantity could be read, without allocating an object. Crucial for security and preventing parsing errors. |

## Integration Patterns

### Standard Usage
ItemQuantity is almost never used in isolation. It is intended to be composed within larger packet objects. The parent packet is responsible for invoking the serialization or deserialization logic.

```java
// A hypothetical packet's deserialization logic
// The packet reads its own fields, then delegates to ItemQuantity
public void deserialize(ByteBuf buf) {
    this.transactionId = buf.readIntLE();
    this.item = ItemQuantity.deserialize(buf, buf.readerIndex());
    buf.skipBytes(ItemQuantity.computeBytesConsumed(buf, buf.readerIndex()));
}

// Game logic processing a received item
public void handleItem(ItemQuantity item) {
    if ("hytale:torch".equals(item.itemId)) {
        player.giveTorches(item.quantity);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Never attempt to deserialize data from an untrusted source without first calling *validateStructure*. Bypassing this step can expose the application to buffer overflow reads or ProtocolExceptions from malformed packets.
- **Manual Deserialization:** Do not attempt to read the fields manually from a ByteBuf. The binary format, especially the *nullBits* field and VarInt encoding, is complex. Always use the provided *deserialize* method.
- **Long-Term Storage:** Do not hold references to ItemQuantity objects for extended periods. They are designed as transient DTOs for network communication, not as a persistent representation of an item stack in the game state.

## Data Pipeline
ItemQuantity acts as a codec at the boundary between raw network bytes and structured game data.

> **Outbound Flow (Serialization):**
> Game State Change -> New **ItemQuantity** instance -> Packet.serialize() calls **ItemQuantity.serialize()** -> Writes to Netty ByteBuf -> Network Socket

> **Inbound Flow (Deserialization):**
> Network Socket -> Netty ByteBuf -> Packet.deserialize() calls **ItemQuantity.deserialize()** -> New **ItemQuantity** instance -> Game Logic Processing

