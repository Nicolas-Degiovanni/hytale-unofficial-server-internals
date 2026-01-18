---
description: Architectural reference for ChangeActiveSlotInteraction
---

# ChangeActiveSlotInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ChangeActiveSlotInteraction extends Interaction {
```

## Architecture & Concepts

The ChangeActiveSlotInteraction class is a **Protocol Data Unit (PDU)**, representing a specific, serializable game event. It is not a service or manager; it is a pure data-transfer object (DTO) that encapsulates the state required to communicate a player's intent to switch their active inventory slot between a client and server.

This class is a concrete implementation of the base Interaction type, which defines a common structure for player-driven actions within the game world. Its primary architectural role is to serve as a strict data contract for network communication. The class contains the complete logic for its own serialization and deserialization into a custom binary format, ensuring that both endpoints of a network connection agree on the data layout.

The serialization format is highly optimized for performance and low bandwidth usage. It employs several advanced techniques:
*   **Fixed/Variable Block Layout:** The serialized data is split into two sections. A fixed-size header contains primitive types and offsets. These offsets point to the locations of variable-sized data (like maps or arrays) in a subsequent data block. This avoids sequential parsing and allows for efficient data access.
*   **Nullability Bitmask:** The first byte of the payload is a bitmask. Each bit corresponds to a nullable complex field (e.g., effects, rules, camera). If the bit is set, the field is present in the data stream; otherwise, it is omitted entirely, saving significant space.
*   **Variable-Length Integers:** It uses VarInt encoding for array and map sizes, which uses fewer bytes to represent smaller numbers.

This object is fundamentally a piece of data in motion, acting as the payload within the protocol layer.

## Lifecycle & Ownership

-   **Creation:** An instance of ChangeActiveSlotInteraction is created in one of two scenarios:
    1.  **Outbound Path (Sending):** Game logic instantiates the object using its constructor when a player performs the corresponding action. For example, the client's input system would create this object when the player scrolls their mouse wheel or presses a hotbar key.
    2.  **Inbound Path (Receiving):** The network layer instantiates the object by calling the static `deserialize` factory method. This occurs when a raw byte buffer from the network is identified as a ChangeActiveSlotInteraction packet.

-   **Scope:** The object's lifetime is exceptionally short and tied to a single network event. On the sending side, it exists only long enough to be passed to the serialization pipeline. On the receiving side, it exists only for the duration of the network handler's execution, after which it is typically discarded.

-   **Destruction:** The object holds no native resources and is managed entirely by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced by the network or game logic thread.

## Internal State & Concurrency

-   **State:** The object is **fully mutable**. Its fields are public and intended to be populated after construction. This design is common for DTOs that are assembled by one system before being serialized and sent to another.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit external synchronization. It is designed for use within a single-threaded context, such as a Netty I/O worker thread or the main game logic thread.

    **WARNING:** Modifying a ChangeActiveSlotInteraction instance from one thread while another thread is serializing or reading it will result in data corruption, serialization errors, and undefined behavior. Treat instances as thread-local.

## API Surface

The primary contract of this class is its static serialization interface and its instance-based serialization method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ChangeActiveSlotInteraction | O(N) | **(Static)** Constructs an object by reading from a byte buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided byte buffer. Returns the number of bytes written. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **(Static)** Performs a safe, read-only check of the buffer to ensure structural integrity before attempting a full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will occupy when serialized. Useful for pre-allocating buffers. |
| clone() | ChangeActiveSlotInteraction | O(N) | Creates a deep copy of the object and its contents. |

*N represents the total size of the variable-length fields (effects, settings, rules, tags, camera).*

## Integration Patterns

### Standard Usage

**Receiving and Processing a Packet**
The object is created by the network layer from an incoming buffer and immediately processed by a handler.

```java
// In a Netty ChannelInboundHandler or similar network dispatcher
void handlePacket(ByteBuf buffer) {
    // Assume packet ID has been read and identifies this type
    ChangeActiveSlotInteraction interaction = ChangeActiveSlotInteraction.deserialize(buffer, buffer.readerIndex());
    
    // Pass the immutable data object to the game thread for processing
    gameLogic.processInteraction(interaction);
}
```

**Creating and Sending a Packet**
Game logic creates and populates the object before handing it to the network layer for serialization.

```java
// In a client-side input handler
void onPlayerChangedSlot(int newSlot) {
    ChangeActiveSlotInteraction interaction = new ChangeActiveSlotInteraction();
    interaction.targetSlot = newSlot;
    // ... set other interaction properties ...
    
    networkManager.sendPacket(interaction); // networkManager will call interaction.serialize()
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** Do not cache and reuse ChangeActiveSlotInteraction objects. They are lightweight DTOs and should be created for each distinct network event to prevent state leakage and bugs related to stale data.

-   **Cross-Thread Modification:** Never modify an instance from a different thread than the one that created it or is processing it. This will lead to severe concurrency issues.

-   **Skipping Validation:** On a server, never call `deserialize` on a buffer received from a client without first calling `validateStructure`. Bypassing this check exposes the server to malformed packets that could trigger exceptions and potentially lead to denial-of-service vulnerabilities.

## Data Pipeline

The flow of this object demonstrates its role as a data container moving between logical layers of the application.

**Outbound Flow (e.g., Client Sending)**
> Player Input -> Input Handling System -> **new ChangeActiveSlotInteraction()** -> Network Dispatcher -> `serialize(ByteBuf)` -> Netty Channel Pipeline -> Raw TCP Packet

**Inbound Flow (e.g., Server Receiving)**
> Raw TCP Packet -> Netty Channel Pipeline -> Packet Frame Decoder -> Packet ID Dispatcher -> `ChangeActiveSlotInteraction.deserialize(ByteBuf)` -> **ChangeActiveSlotInteraction instance** -> Game Logic Handler -> Player State Update

