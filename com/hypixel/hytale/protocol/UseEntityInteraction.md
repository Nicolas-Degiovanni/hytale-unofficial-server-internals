---
description: Architectural reference for UseEntityInteraction
---

# UseEntityInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class UseEntityInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The UseEntityInteraction class is a Data Transfer Object (DTO) that represents a specific, structured command within the Hytale network protocol. It is not a service or manager; its sole purpose is to encapsulate the data defining how a player character interacts with an in-game entity. This class acts as a data contract between the client and server, ensuring both sides agree on the structure and meaning of an interaction event.

Architecturally, this class is a critical component of the network serialization layer. It employs a highly optimized custom binary format to minimize network bandwidth and reduce processing overhead. The serialization strategy is notable for its two-part structure:

1.  **Fixed-Size Block:** A contiguous block of 19 bytes containing primitive data types like floats, integers, and booleans. This allows for extremely fast, direct memory access.
2.  **Variable-Size Block:** A subsequent data block for complex, nullable, or variable-length fields such as maps, arrays, and nested objects. The fixed-size block contains integer offsets that point to the location of these variable fields within the buffer.

To further optimize for size, the first byte of the entire serialized object is a **nullBits** bitfield. Each bit in this byte corresponds to a nullable, variable-sized field, indicating whether it is present in the data stream. This avoids the need to write placeholder data for absent optional fields.

## Lifecycle & Ownership
- **Creation:** An instance of UseEntityInteraction is created in one of two ways:
    1.  **Deserialization:** The static factory method `deserialize` is called by a network protocol handler (e.g., a Netty pipeline handler) to construct an object from an incoming network ByteBuf. This is the most common creation path for received data.
    2.  **Direct Instantiation:** Game logic systems create a new instance via its constructor when a local player action needs to be serialized and sent over the network.
- **Scope:** This object is ephemeral and has a very short lifecycle. Its scope is typically bound to the processing of a single network packet or the duration of a single game tick.
- **Destruction:** The object holds no managed resources and is intended to be garbage collected as soon as the network event has been fully processed by the relevant game systems.

## Internal State & Concurrency
- **State:** The object is fully **mutable**. Its fields are populated during deserialization or can be set directly after construction. It is effectively a container for data.
- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. Concurrent reads and writes from multiple threads without external locking mechanisms will result in data corruption and undefined behavior.

**WARNING:** Do not share instances of this class across threads. If data must be passed to another thread, create a deep copy using the `clone` method.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a data marshalling utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | UseEntityInteraction | O(N) | **[Static]** Constructs an object from a binary representation in a buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided buffer using the custom binary format. Returns bytes written. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | **[Static]** Calculates the total size of a serialized object in a buffer without full deserialization. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **[Static]** Performs pre-flight checks on a buffer to ensure data is well-formed before attempting deserialization. |
| clone() | UseEntityInteraction | O(N) | Creates a deep copy of the object, including all nested collections and objects. |

## Integration Patterns

### Standard Usage
The primary use case involves a protocol handler decoding a network buffer into a usable object for game logic systems.

```java
// Within a network message handler
public void handlePacket(ByteBuf packetData) {
    // Before processing, validate the structure to prevent errors
    ValidationResult result = UseEntityInteraction.validateStructure(packetData, packetData.readerIndex());
    if (!result.isValid()) {
        // Handle corrupted or malicious packet
        throw new ProtocolException("Invalid UseEntityInteraction packet: " + result.error());
    }

    // Deserialize into a concrete object
    UseEntityInteraction interaction = UseEntityInteraction.deserialize(packetData, packetData.readerIndex());

    // Pass the structured data to the game's interaction system
    game.getInteractionSystem().process(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Never call `deserialize` on a buffer received from an external source without first calling `validateStructure`. Failing to do so can result in uncaught exceptions and potential server or client instability if the data is malformed.
- **Instance Reuse:** Do not modify a deserialized instance and attempt to serialize it for re-transmission. These objects are cheap to create; instantiate a new object to ensure a clean state.
- **Concurrent Modification:** Do not pass an instance to another thread while retaining a reference. The lack of thread safety makes this pattern extremely dangerous.

## Data Pipeline
The UseEntityInteraction class serves as a translation point between raw byte streams and structured game data.

> Flow (Incoming Packet):
> Raw Network ByteBuf -> Netty Channel Handler -> **UseEntityInteraction.deserialize** -> UseEntityInteraction Instance -> Game Logic System -> Entity State Change

