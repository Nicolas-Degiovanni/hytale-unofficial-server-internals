---
description: Architectural reference for UpdateServerPlayerList
---

# UpdateServerPlayerList

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class UpdateServerPlayerList implements Packet {
```

## Architecture & Concepts
The UpdateServerPlayerList class is a Data Transfer Object (DTO) that represents a network packet. It is a concrete implementation of the Packet interface, designed for one specific purpose: to synchronize the state of the server-side player list with the client. This is commonly used to populate and update the user interface element that displays all connected players (often called the "tab list").

This class acts as a data contract between the server and client. Its structure is defined by the Hytale network protocol and is not intended for general application logic. The class encapsulates the serialization and deserialization logic required to convert its state to and from the raw byte stream managed by the Netty network framework. It is a low-level component, sitting at the boundary between the raw network pipeline and higher-level game state management systems.

## Lifecycle & Ownership
-   **Creation:**
    -   **Outbound (Server):** Instantiated directly via its constructor (`new UpdateServerPlayerList(...)`) by a server-side system that monitors player connections and status changes.
    -   **Inbound (Client):** Instantiated by the network protocol layer via the static factory method **deserialize** when a raw network buffer with Packet ID 226 is received.
-   **Scope:** This object is extremely short-lived. It exists only for the brief moment of its creation, serialization/deserialization, and processing by a single packet handler. It is a transient, single-use object.
-   **Destruction:** The object is eligible for garbage collection immediately after the corresponding packet handler has finished processing it. There are no manual cleanup or resource release mechanisms.

## Internal State & Concurrency
-   **State:** The primary state is a single, nullable array of ServerPlayerListUpdate objects named **players**. This state is fully mutable and directly represents the data to be transmitted. It does not cache any information and its contents are overwritten upon deserialization.
-   **Thread Safety:** **This class is not thread-safe.** Its internal array is mutable and directly exposed. This design assumes that packet processing occurs on a single, dedicated network thread (e.g., a Netty EventLoop).

    **WARNING:** Never share an instance of this packet across multiple threads without explicit synchronization. If data from this packet must be passed to another thread (such as the main game thread), it is critical to either copy the data into a thread-safe data structure or use a queuing mechanism to ensure safe handoff.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | UpdateServerPlayerList | O(N) | Static factory. Deserializes the packet from a ByteBuf. N is the number of players in the list. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Serializes the packet's state into a ByteBuf for network transmission. N is the number of players. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the exact number of bytes consumed by a packet instance within a buffer. N is the number of players. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Performs a fast, shallow check on the buffer to validate packet headers and declared sizes without full deserialization. |
| clone() | UpdateServerPlayerList | O(N) | Performs a deep copy of the packet, creating new ServerPlayerListUpdate instances for the internal array. |

## Integration Patterns

### Standard Usage
This packet is processed by a client-side packet handler. The handler extracts the player data and forwards it to a UI or game state manager responsible for maintaining the player list.

```java
// In a client-side packet handler
public void handle(UpdateServerPlayerList packet) {
    // Retrieve the service responsible for the UI player list
    PlayerListManager manager = context.getService(PlayerListManager.class);

    // Pass the data for processing. The manager is responsible for
    // applying the updates to the game state on the correct thread.
    if (packet.players != null) {
        manager.applyUpdates(packet.players);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Modification After Deserialization:** Do not modify the contents of the **players** array after the packet has been deserialized. Treat the object as immutable once it has been created by the network layer to avoid unpredictable side effects in other potential listeners.
-   **Cross-Thread Access:** Do not read the **players** field from a different thread than the network thread that deserialized it. This will lead to race conditions. Dispatch the work or data to the target thread instead.
-   **Reusing Instances:** Do not attempt to reuse a packet instance for sending multiple updates. Always create a new instance for each distinct state change to ensure data integrity.

## Data Pipeline
The UpdateServerPlayerList packet is a key component in the flow of player state information from the server to the client's user interface.

> **Outbound Flow (Server):**
> Player Connection Event -> Server Player Manager -> **new UpdateServerPlayerList()** -> Netty Channel Pipeline -> **serialize()** -> TCP/IP Stack

> **Inbound Flow (Client):**
> TCP/IP Stack -> Netty Channel Pipeline -> Packet Decoder -> **UpdateServerPlayerList.deserialize()** -> Client Packet Handler -> Player List Manager -> UI Render Update

