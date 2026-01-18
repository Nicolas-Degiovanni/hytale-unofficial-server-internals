---
description: Architectural reference for FirstClickInteraction
---

# FirstClickInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class FirstClickInteraction extends Interaction {
```

## Architecture & Concepts

The FirstClickInteraction class is a concrete Data Transfer Object (DTO) that represents the serialized state of a player's initial interaction with an in-game element. It serves as a strict data contract within the Hytale network protocol, defining the precise binary layout for "first click" or "press" type events. This class is not a service or manager; its sole responsibility is to encapsulate data for network transmission and provide the logic for its own serialization and deserialization.

Architecturally, it is a fundamental component of the network layer's data model. Its design prioritizes performance and network efficiency through a custom binary format, rather than relying on generic serialization frameworks. The binary structure is composed of two primary sections:

1.  **Fixed-Size Block:** A 19-byte block at the beginning of the object's data containing primitive fields like click, held, and runTime. This allows for constant-time access to core data.
2.  **Variable-Size Block:** A subsequent block containing complex, nullable, or variable-length data structures such as InteractionEffects, settings maps, and tags arrays. The fixed-size block contains integer offsets pointing to the start of each corresponding structure within this variable block.

A single byte, referred to as *nullBits*, acts as a bitfield at the very start of the serialized data. Each bit corresponds to a nullable field, indicating its presence or absence in the data stream. This avoids the need to write null markers for every optional field, significantly reducing payload size.

## Lifecycle & Ownership

-   **Creation:** An instance of FirstClickInteraction is almost exclusively created by the network protocol layer. On the receiving end (e.g., a game server), the static factory method **deserialize** is invoked by a packet dispatcher to construct the object from a raw Netty ByteBuf. On the sending end, an instance may be created using its constructor, populated with data, and passed to the network layer for serialization.

-   **Scope:** The object's lifetime is extremely short and tied to the scope of a single network packet's processing cycle. It is created, read by a handler, and then immediately becomes eligible for garbage collection.

-   **Destruction:** There is no explicit destruction or resource cleanup. Instances are managed by the Java Garbage Collector. Due to their transient nature, they exert minimal pressure on memory management systems.

## Internal State & Concurrency

-   **State:** The internal state is fully mutable. All fields are public and are populated directly during the deserialization process. The object is designed to be a simple data container and does not enforce immutability.

-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives or atomic operations. It is designed to be created, processed, and discarded within the confines of a single thread, typically a Netty I/O worker thread or the main game logic thread.

    **WARNING:** Sharing an instance of FirstClickInteraction across multiple threads without external locking will lead to unpredictable behavior, data corruption, and race conditions. Do not store instances in shared collections or pass them between thread contexts.

## API Surface

The public contract is dominated by static methods for serialization and validation, reflecting its role as a protocol utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | FirstClickInteraction | O(N) | **Primary Constructor.** Reads from a ByteBuf at a given offset and constructs a new instance. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the binary data in a buffer without full deserialization. Critical for preventing buffer overflows or parsing malformed packets. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object within a buffer by reading its headers and variable-length field metadata. |
| clone() | FirstClickInteraction | O(N) | Creates a deep copy of the object and its nested data structures. |

*Complexity O(N) refers to the total size of the variable-length data fields.*

## Integration Patterns

### Standard Usage

The class should be used by a network packet handler to decode an incoming buffer into a usable Java object, which is then passed to the game logic layer.

```java
// In a network handler receiving a ByteBuf
public void handlePacket(ByteBuf packetData) {
    // Assume packet ID has been read and we know the type
    int objectOffset = packetData.readerIndex();

    ValidationResult result = FirstClickInteraction.validateStructure(packetData, objectOffset);
    if (!result.isValid()) {
        // Disconnect client or log error
        throw new ProtocolException("Invalid FirstClickInteraction: " + result.error());
    }

    FirstClickInteraction interaction = FirstClickInteraction.deserialize(packetData, objectOffset);

    // Pass the fully formed object to the game event system
    game.getEventBus().post(new PlayerInteractionEvent(interaction));
}
```

### Anti-Patterns (Do NOT do this)

-   **Object Re-use:** Do not hold onto instances of FirstClickInteraction after processing. They represent a point-in-time snapshot of an event and should be discarded. Re-using or modifying them for subsequent packets is not safe.
-   **Multi-threaded Access:** As stated in the concurrency section, never access a single instance from multiple threads. If data must be passed to another thread, create a new, thread-safe object containing only the required information.
-   **Manual Deserialization:** Do not attempt to read the fields from the ByteBuf manually. The binary layout is complex, and any deviation from the provided deserialize and serialize methods will lead to data corruption.

## Data Pipeline

The FirstClickInteraction object is a payload carrier in the network data pipeline. It represents the transformation of raw bytes into a structured, domain-specific object.

> Flow:
> Network ByteBuf -> Protocol Dispatcher -> **FirstClickInteraction.deserialize** -> Game Event Bus -> Interaction Handler Logic

