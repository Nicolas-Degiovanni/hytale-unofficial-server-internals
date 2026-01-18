---
description: Architectural reference for SetGameMode
---

# SetGameMode

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient

## Definition
```java
// Signature
public class SetGameMode implements Packet {
```

## Architecture & Concepts
The SetGameMode class is a Data Transfer Object (DTO) that represents a specific, atomic command within the Hytale network protocol. Its sole purpose is to encapsulate the data required to change a player's current game mode, such as switching between Adventure and Creative modes.

As an implementation of the Packet interface, this class is a fundamental unit of communication between the client and server. It is not a service or a manager; rather, it is the payload processed by the network pipeline and game logic handlers. Its design is optimized for high-performance serialization and deserialization directly to and from a Netty ByteBuf, minimizing memory allocation and processing overhead. The class structure is rigid and corresponds directly to a predefined binary layout on the wire, identified by the static constant PACKET_ID 101.

## Lifecycle & Ownership
- **Creation:** An instance of SetGameMode is created under two distinct circumstances:
    1. **Inbound (Deserialization):** The network layer's packet dispatcher invokes the static factory method SetGameMode.deserialize when it reads a ByteBuf with a packet identifier of 101. This is the most common creation path on the receiving end (e.g., a client receiving a command from the server).
    2. **Outbound (Game Logic):** Game logic on the sending end (e.g., a server responding to a command or event) instantiates the class directly using its constructor, `new SetGameMode(GameMode.Creative)`, before passing it to the network encoder for serialization.

- **Scope:** The object's lifetime is exceptionally brief and transactional. It is intended to be a short-lived, "fire-and-forget" data container. It exists only for the duration of its journey through a single processing tick or network event handler.

- **Destruction:** The object becomes eligible for garbage collection immediately after it has been serialized to a buffer or after its data has been consumed by a game logic handler. There is no manual cleanup mechanism; ownership is never transferred or persisted.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: the *gameMode* field. It is a simple data holder with no caching, lazy initialization, or complex internal logic. Its state is fully defined at the moment of creation.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed to be created, processed, and discarded within the confines of a single thread, typically a Netty I/O worker thread or a main game loop thread. Concurrent modification of the gameMode field would lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | SetGameMode | O(1) | Static factory. Reads one byte from the buffer to construct a new instance. |
| serialize(ByteBuf) | void | O(1) | Writes the single byte value of the game mode into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload, which is always 1 byte. |
| getId() | int | O(1) | Returns the static network identifier for this packet type, 101. |

## Integration Patterns

### Standard Usage
The SetGameMode packet is almost exclusively handled by the core protocol engine. Game logic should not interact with its serialization methods directly. Instead, it either creates an instance to be sent or receives an instance within a registered event handler.

```java
// Example of a server sending the packet
GameMode newMode = GameMode.Creative;
SetGameMode packet = new SetGameMode(newMode);
playerConnection.sendPacket(packet);

// Example of a client-side handler receiving the packet
public void onSetGameMode(SetGameMode packet) {
    this.localPlayer.setGameMode(packet.gameMode);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching or Re-use:** Do not cache and re-send a SetGameMode instance. Packets are cheap to create and should be treated as immutable messages once created. Re-using instances can lead to unpredictable behavior in the network pipeline.
- **Cross-Thread Access:** Never create a SetGameMode instance on one thread and pass its reference to another for modification or reading without proper synchronization. This violates the design contract.
- **Manual Serialization:** Avoid calling serialize or deserialize outside of the dedicated network channel handlers. The protocol layer is responsible for managing the binary lifecycle of packets.

## Data Pipeline
The flow of data for this packet is linear and unidirectional within a single transaction.

> **Outbound Flow (Server -> Client):**
> Server Game Logic -> `new SetGameMode()` -> Network Encoder -> `serialize(ByteBuf)` -> TCP Socket

> **Inbound Flow (Client Receives):**
> TCP Socket -> Netty ByteBuf -> Packet Dispatcher (reads ID 101) -> `SetGameMode.deserialize(ByteBuf)` -> **SetGameMode Instance** -> Client-side Event Handler -> Player State Update

