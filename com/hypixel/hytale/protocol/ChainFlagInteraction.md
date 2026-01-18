---
description: Architectural reference for ChainFlagInteraction
---

# ChainFlagInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class ChainFlagInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The ChainFlagInteraction class is a specialized Data Transfer Object (DTO) within the Hytale network protocol layer. It represents a concrete, serializable state for a specific type of game interaction. As a subclass of SimpleInteraction, it inherits a baseline set of interaction properties and adds its own unique fields, **chainId** and **flag**, to support sequenced or conditional gameplay events.

Its primary architectural role is to serve as a passive data container that defines the precise wire format for communication between the client and server. The class encapsulates the complex logic for serializing its state into a Netty ByteBuf and deserializing from a ByteBuf back into a Java object.

The serialization strategy is highly optimized for network performance and payload size. It employs a hybrid fixed-variable block layout:
1.  **Nullable Bit Field:** The first byte is a bitmask indicating which of the seven nullable, variable-sized fields are present in the payload. This avoids wasting bytes for absent optional data.
2.  **Fixed Block:** A 19-byte block containing primitive fields that are always present, such as runTime and next.
3.  **Offset Table:** A 28-byte block (7 fields * 4 bytes/offset) containing integer offsets. Each offset points to the start of a corresponding variable-sized field's data within the variable block.
4.  **Variable Block:** A contiguous block of memory appended after the fixed block and offset table, containing the actual data for all present nullable fields (e.g., serialized strings, maps, and nested objects).

This structure allows for extremely fast, direct-access reads of fixed-size data while efficiently handling variable-sized and optional data.

## Lifecycle & Ownership
- **Creation:**
    - **Inbound:** An instance is created exclusively by the network protocol layer when a corresponding packet is received. The static factory method **deserialize** is invoked by a packet dispatcher which reads from an incoming ByteBuf.
    - **Outbound:** Game logic instantiates this class directly using its constructor when it needs to send this specific interaction to a remote endpoint. The instance is then passed to the network layer for serialization.
- **Scope:** The object is **transient** and short-lived. Its lifecycle is bound to the processing of a single network packet. Once the packet data has been consumed by the relevant game system, the object is no longer referenced.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it falls out of the scope of the network handler or game logic method that was processing it.

## Internal State & Concurrency
- **State:** The object is **mutable**. Its public fields are intended to be populated during deserialization or set by game logic prior to serialization. It is a plain data holder and does not cache any information or manage external resources.
- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single thread, such as a Netty I/O worker thread or the main game loop thread. Concurrent modification from multiple threads without external locking mechanisms will result in a corrupted state and is strictly unsupported.

## API Surface
The public API is dominated by static methods for creating and validating instances from raw byte buffers, reflecting its role as a protocol-defined data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ChainFlagInteraction | O(N) | Constructs an object by reading from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state to a buffer and returns the number of bytes written. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(k) | Performs a structural pre-validation of the data in a buffer without full deserialization. Checks offsets and lengths. |
| computeBytesConsumed(ByteBuf, int) | static int | O(k) | Calculates the total size of a serialized object within a buffer. Used for skipping over packets. |
| clone() | ChainFlagInteraction | O(N) | Creates a deep copy of the object. |

*N = total size of variable data fields; k = number of present variable fields*

## Integration Patterns

### Standard Usage
The canonical use case involves a network handler deserializing the object from a buffer and passing it to a game system for processing.

```java
// Executed within a network thread or game logic handler
void handleInteractionPacket(ByteBuf packetData) {
    // The offset would be determined by the packet framing protocol
    int payloadOffset = ...;

    // Validate before deserializing to prevent errors
    if (!ChainFlagInteraction.validateStructure(packetData, payloadOffset).isValid()) {
        throw new InvalidPacketException("Malformed ChainFlagInteraction");
    }

    ChainFlagInteraction interaction = ChainFlagInteraction.deserialize(packetData, payloadOffset);

    // Pass the fully-formed object to the game's interaction system
    game.getInteractionSystem().process(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold onto and modify a deserialized instance to send a new packet. The object's state is not designed to be reset. Always create a new instance for outbound data to ensure a clean state.
- **Concurrent Modification:** Do not share an instance between threads. If data must be passed to another system on a different thread, either create a full copy using the **clone** method or extract the primitive data into a dedicated, thread-safe structure.
- **Manual Buffer Manipulation:** Do not attempt to manually read or write fields to a ByteBuf to mimic this object's structure. The serialization format is complex, involving offsets and bitmasks. Always use the provided **serialize** and **deserialize** methods to guarantee protocol compliance.

## Data Pipeline
The ChainFlagInteraction class is a critical link in the network data pipeline, translating raw bytes into a structured, usable game object and back again.

> **Inbound Flow:**
> Raw TCP Stream -> Netty ByteBuf -> Protocol Dispatcher -> **ChainFlagInteraction.deserialize()** -> Game Logic System

> **Outbound Flow:**
> Game Logic System -> **new ChainFlagInteraction()** -> Protocol Encoder -> **instance.serialize()** -> Netty ByteBuf -> Raw TCP Stream

