---
description: Architectural reference for RemoveFromServerPlayerList
---

# RemoveFromServerPlayerList

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class RemoveFromServerPlayerList implements Packet {
```

## Architecture & Concepts
The RemoveFromServerPlayerList class is a Data Transfer Object (DTO) that represents a specific network message within the Hytale protocol. Its sole purpose is to encapsulate the data required to instruct a client to remove one or more players from its local representation of the server's player list.

As an implementation of the Packet interface, this class is a fundamental component of the network serialization layer. It contains no game logic. Instead, it provides a strongly-typed interface for serializing data into a Netty ByteBuf for network transmission and deserializing from a ByteBuf upon receipt.

The binary layout is optimized for network efficiency, utilizing a single bit in a bitmask to indicate the presence of the nullable player list, and a VarInt to encode the array length. This design minimizes packet size, which is critical for performance in a real-time game environment.

## Lifecycle & Ownership
- **Creation:**
    - **Receiving End (Client):** Instantiated exclusively by the protocol's deserialization pipeline. A central Netty channel handler reads the packet ID (225) from the inbound ByteBuf and invokes the static `deserialize` factory method to construct the object.
    - **Sending End (Server):** Instantiated directly via `new RemoveFromServerPlayerList(UUID[])` by a higher-level game state manager, such as a SessionManager or WorldManager, in response to a player disconnection event.

- **Scope:** Extremely short-lived. An instance of this class exists only for the brief duration of its processing within a single network event loop tick. It is a message, not a persistent entity.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by a game system (on the client) or after it has been serialized into the outbound network buffer (on the server). There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Mutable. The primary state is the public `players` array. While technically mutable, instances of this class should be treated as immutable after creation. Modifying the state of a packet after it has been queued for sending or during its processing by a handler is an anti-pattern that can lead to undefined behavior.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded within the confines of a single thread, typically a Netty EventLoop. Do not share instances of this packet across threads without explicit, external synchronization. The public, non-final `players` field makes it inherently unsafe for concurrent access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | RemoveFromServerPlayerList | O(N) | **Static Factory.** Constructs a new instance by reading from a network buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided network buffer for transmission. |
| computeSize() | int | O(1) | Calculates the exact number of bytes required to serialize this packet. Crucial for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Performs a lightweight, non-deserializing check on a buffer to validate the packet structure. |

*N represents the number of UUIDs in the `players` array.*

## Integration Patterns

### Standard Usage
This packet is processed by a network event handler which dispatches the data to a system responsible for managing the client-side player list.

```java
// On the receiving end (e.g., ClientPacketHandler)
// packet is an instance of RemoveFromServerPlayerList
PlayerListManager manager = context.getService(PlayerListManager.class);
if (packet.players != null) {
    for (UUID playerId : packet.players) {
        manager.removePlayer(playerId);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify the `players` array after the packet has been passed to the network layer for sending. The serialization process may not be immediate, leading to data races.
- **Cross-Thread Sharing:** Do not access a packet instance from a different thread than the one that created it (e.g., the Netty EventLoop). This will lead to memory visibility issues and race conditions.
- **Reusing Instances:** Do not attempt to reuse packet objects. They are cheap to create and should be instantiated for each message.

## Data Pipeline

> **Inbound Flow (Client):**
> Raw TCP Socket -> Netty Frame Decoder -> Packet Deserializer -> **RemoveFromServerPlayerList** -> Client Game Event Bus -> PlayerListUI / PlayerManager

> **Outbound Flow (Server):**
> Player Disconnect Event -> Session Manager -> `new RemoveFromServerPlayerList()` -> Packet Serializer -> Netty Frame Encoder -> Raw TCP Socket

