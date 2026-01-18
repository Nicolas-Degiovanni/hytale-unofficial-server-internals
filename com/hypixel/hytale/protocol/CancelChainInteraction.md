---
description: Architectural reference for CancelChainInteraction
---

# CancelChainInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class CancelChainInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The CancelChainInteraction class is a **Protocol Data Unit (PDU)**, representing a specific message within Hytale's client-server communication protocol. It is not a service or manager, but rather a structured data container that defines the binary layout for a command to terminate a sequence of player actions, known as an "interaction chain".

Its primary architectural function is to serve as the boundary object between raw network byte streams and the game's interaction logic. The class encapsulates the complex serialization and deserialization logic required to translate its state to and from the Netty ByteBuf format used by the networking layer.

The binary layout is highly optimized for network efficiency, employing a hybrid structure:
1.  **Fixed-Size Block:** A 19-byte block at the beginning of the payload contains primitive fields that are always present, such as multipliers, flags, and state identifiers.
2.  **Offset Table:** A 24-byte block follows the fixed data, containing integer offsets pointing to the location of variable-sized data.
3.  **Variable-Size Block:** All complex, nullable, or variable-length data (such as nested objects, maps, and arrays) is appended at the end of the message. The offset table ensures the parser can locate this data without scanning the payload.
4.  **Null Bitmask:** The very first byte of the message is a bitmask (nullBits). Each bit corresponds to a nullable, variable-sized field. If a bit is set, the corresponding field is present in the payload and its offset in the table is valid. This is a critical optimization that avoids transmitting any data for optional fields that are not in use.

## Lifecycle & Ownership

-   **Creation:** An instance of CancelChainInteraction is created under two distinct circumstances:
    1.  **On the sending endpoint (Client or Server):** Game logic instantiates the object directly using its constructor, populates its fields, and passes it to the protocol serialization pipeline.
    2.  **On the receiving endpoint:** The static factory method **deserialize** is invoked by a central packet dispatcher. This method allocates a new CancelChainInteraction object and populates it by reading directly from the network ByteBuf.
-   **Scope:** The object is **transient** and has an extremely short lifetime. Its scope is typically confined to a single network event processing cycle. It exists solely to transport data from the network layer to the relevant game logic handler.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the consuming system (e.g., an interaction event handler) has finished processing it. No long-term references are maintained.

## Internal State & Concurrency

-   **State:** The internal state is fully **Mutable**. All data-holding fields are public and intended to be directly manipulated after creation or deserialization. This class is a pure data structure and holds no derived or cached state.
-   **Thread Safety:** This class is **not thread-safe**. It contains no locks or other concurrency primitives. It is designed under the assumption that it will be created, processed, and discarded within a single thread, such as a Netty event loop thread or the main game thread.

**WARNING:** Sharing an instance of CancelChainInteraction across multiple threads without external synchronization will lead to race conditions and undefined behavior. The standard pattern is to hand off ownership of the object from the network thread to a game logic thread via a thread-safe queue.

## API Surface

The public contract is dominated by static methods for network I/O and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CancelChainInteraction | O(N) | **[Factory]** Constructs a new object from a network buffer. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided network buffer. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a structurally valid message. Does not instantiate the object. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object within a buffer by reading its headers and variable-length field sizes. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. Used for buffer pre-allocation. |

*N represents the total size of variable-length fields (collections, strings, nested objects).*

## Integration Patterns

### Standard Usage

The class is almost exclusively used by the network protocol layer. A packet handler receives a buffer, identifies the packet type, and uses the static deserialize method to construct the object before passing it to a higher-level system.

```java
// Executed by a packet handler on a network thread
public void handlePacket(ByteBuf packetBuffer) {
    // Validate packet structure to prevent parsing errors
    ValidationResult result = CancelChainInteraction.validateStructure(packetBuffer, 0);
    if (!result.isValid()) {
        // Disconnect client or log error
        return;
    }

    // Deserialize the full object
    CancelChainInteraction interaction = CancelChainInteraction.deserialize(packetBuffer, 0);

    // Dispatch the data object to the game logic thread for processing
    gameLogicQueue.post(interaction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Object Re-use:** Do not attempt to pool or reuse CancelChainInteraction instances. They are lightweight objects designed to be created and discarded for each message. Reusing them can lead to subtle state corruption.
-   **Manual Deserialization:** Do not attempt to read fields from the ByteBuf manually. The binary format is complex, involving bitmasks and relative offsets. Always use the provided **deserialize** method.
-   **Cross-Thread Modification:** Never modify an instance from one thread while another thread might be reading it. For example, do not read from the object on the main game thread while the network thread is still deserializing it.

## Data Pipeline

The CancelChainInteraction object is a key component in the network data flow, acting as the structured representation of a raw byte payload.

> **Receiving Flow:**
> Raw TCP Packet -> Netty Channel Pipeline -> Packet Frame Decoder -> **CancelChainInteraction.deserialize** -> Game Event Bus -> Interaction System

> **Sending Flow:**
> Interaction System -> `new CancelChainInteraction()` -> **CancelChainInteraction.serialize** -> Packet Frame Encoder -> Netty Channel Pipeline -> Raw TCP Packet

