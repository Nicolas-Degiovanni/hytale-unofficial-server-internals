---
description: Architectural reference for UpdateServerPlayerListPing
---

# UpdateServerPlayerListPing

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class UpdateServerPlayerListPing implements Packet {
```

## Architecture & Concepts

The UpdateServerPlayerListPing class is a concrete implementation of the Packet interface. It serves as a Data Transfer Object (DTO) within the Hytale network protocol, specifically designed to transmit the ping times for all players connected to a server. Its sole responsibility is to encapsulate this data and provide the logic for its own serialization and deserialization.

This class follows a common pattern in the Hytale protocol library where the data structure and its network codec are colocated. The static methods, such as deserialize, computeBytesConsumed, and validateStructure, act as a self-contained codec. This design avoids the need for separate, external codec classes, ensuring that the binary representation of a packet is always consistent with its in-memory data structure.

The primary data payload is a Map of player UUIDs to their corresponding ping values in milliseconds. The network format is highly optimized, using a bitmask for null fields and variable-length integers (VarInt) for collection sizes to minimize bandwidth.

## Lifecycle & Ownership

-   **Creation:**
    -   **Outbound (Server-Side):** Instantiated by a server-side system responsible for tracking player latency. The system populates the player map and queues the packet for transmission to clients.
    -   **Inbound (Client-Side):** Instantiated by the network protocol layer. When an incoming data stream is identified with Packet ID 227, the protocol dispatcher invokes the static `deserialize` method, which constructs and returns a new UpdateServerPlayerListPing instance.

-   **Scope:** This object is **transient and short-lived**. It exists only for the brief period required to be serialized and sent, or deserialized and processed. It is not intended to be stored long-term.

-   **Destruction:** The object is eligible for garbage collection as soon as it has been processed by its corresponding handler (on the client) or written to the network buffer (on the server). There is no manual destruction logic.

## Internal State & Concurrency

-   **State:** The internal state, primarily the `players` map, is **mutable**. The object is a simple data container. While the fields can be changed after instantiation, this is strongly discouraged once the packet enters the network pipeline.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread. Handing this object between threads (e.g., from a network thread to a main game logic thread) requires external synchronization or the use of thread-safe queues to prevent race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static UpdateServerPlayerListPing | O(N) | Constructs a new instance by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the number of players. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf. Throws ProtocolException if constraints are violated (e.g., map too large). |
| computeSize() | int | O(1) | Calculates the exact number of bytes this packet will occupy on the wire. Does not perform a full serialization. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a safe, read-only check of the buffer to ensure it contains a valid packet structure without fully deserializing. |

## Integration Patterns

### Standard Usage

This packet is typically handled by a client-side system that manages the UI for the player list. The handler extracts the data and updates the relevant UI components.

```java
// Example of a client-side packet handler
public void handlePlayerListPing(UpdateServerPlayerListPing packet) {
    if (packet.players == null) {
        return;
    }

    // Get the UI service or component responsible for the player list
    PlayerListUI playerList = uiContext.getComponent(PlayerListUI.class);

    // Update the UI with the new ping data
    playerList.updatePlayerPings(packet.players);
}
```

### Anti-Patterns (Do NOT do this)

-   **State Mutation After Queuing:** Modifying the `players` map after the packet has been passed to a network manager for sending. This can lead to data races where an incomplete or incorrect state is serialized and transmitted.
-   **Cross-Thread Processing:** Deserializing the packet on a Netty I/O thread and directly accessing its `players` map from the main game thread without a memory barrier or synchronization. The data should be defensively copied or passed through a concurrent queue.
-   **Reusing Instances:** Do not hold onto packet instances for reuse. They are cheap to create and should be treated as immutable once populated.

## Data Pipeline

The flow of data for this packet differs depending on the direction.

> **Outbound Flow (Server to Client):**
> Server Game State -> **UpdateServerPlayerListPing (new instance)** -> `serialize()` -> Netty Channel Pipeline -> Encoded ByteBuf -> Network Socket

> **Inbound Flow (Client from Server):**
> Network Socket -> Raw ByteBuf -> Netty Channel Pipeline -> Packet Framer -> **UpdateServerPlayerListPing.deserialize()** -> Packet Handler -> UI State Update

