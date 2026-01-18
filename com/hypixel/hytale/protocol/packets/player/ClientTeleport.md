---
description: Architectural reference for ClientTeleport
---

# ClientTeleport

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class ClientTeleport implements Packet {
```

## Architecture & Concepts
The ClientTeleport class is a Data Transfer Object (DTO) that represents a server-authoritative command to forcibly relocate a player client. As part of the core Hytale network protocol, it serves as a structured data container for information flowing from the server's game logic to the client's rendering and physics engine.

This packet is a fundamental mechanism for maintaining world consistency, handling events like respawning, using in-game teleporters, or correcting client-side prediction errors. It is a one-way command; the client receives this packet, acts upon it, and may be required to send a confirmation response to the server. Its design prioritizes performance and a minimal memory footprint, featuring a fixed-size binary layout for rapid serialization and deserialization by the Netty pipeline.

## Lifecycle & Ownership
- **Creation:** An instance of ClientTeleport is created exclusively by the client-side network protocol layer. Specifically, a packet deserializer, operating within a Netty I/O thread, instantiates this object upon receiving a byte stream with the corresponding packet ID (109).
- **Scope:** The object's lifetime is exceptionally brief and confined to a single processing tick. It is created, passed to a packet handler, and immediately becomes eligible for garbage collection once the handler completes its execution.
- **Destruction:** There is no explicit destruction logic. The Java Garbage Collector reclaims the memory. Long-term references to this object must not be held, as they would constitute a memory leak.

## Internal State & Concurrency
- **State:** The object's state is mutable, with public fields for direct access. However, it should be treated as an immutable record after deserialization. The state represents a snapshot of the teleport command at the moment it was sent by the server. It contains no caching or derived data.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created on a Netty I/O thread and immediately handed off to a single-threaded game logic loop for processing. Concurrent access from multiple threads will lead to race conditions and undefined behavior. All interactions with this object must be synchronized with the main game thread.

## API Surface
The primary contract is for serialization and deserialization, driven by the network framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ClientTeleport | O(1) | Static factory method. Constructs a ClientTeleport object from a raw Netty ByteBuf. |
| serialize(buf) | void | O(1) | Encodes the object's state into a Netty ByteBuf for network transmission. |
| getId() | int | O(1) | Returns the static network identifier (109) for this packet type. |
| computeSize() | int | O(1) | Returns the fixed size (52 bytes) of the packet on the wire. |

## Integration Patterns

### Standard Usage
The packet is received by the network layer and dispatched to a registered handler, typically on the main game thread. The handler extracts the position and orientation data and applies it directly to the local player entity.

```java
// Within a Packet Handler implementation
public void handle(ClientTeleport packet) {
    Player localPlayer = world.getLocalPlayer();
    
    // Apply the new transform to the player entity
    if (packet.modelTransform != null) {
        localPlayer.applyTransform(packet.modelTransform);
    }

    // Reset physics state if commanded by the server
    if (packet.resetVelocity) {
        localPlayer.getPhysicsBody().resetVelocity();
    }

    // The server may require a confirmation packet
    // connection.send(new ServerTeleportConfirm(packet.teleportId));
}
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** Do not create instances of ClientTeleport on the client using `new`. This packet is exclusively a server-to-client command. Creating it on the client serves no purpose and indicates a misunderstanding of the protocol flow.
- **State Modification:** Do not modify the fields of a received ClientTeleport packet. It represents a server-authoritative command and must be treated as read-only.
- **Asynchronous Processing:** Do not process this packet on a separate thread without proper synchronization. Applying the teleport transform must be atomic with respect to the game's physics and rendering loop to prevent visual tearing or physics glitches.

## Data Pipeline
The data flow for this packet is unidirectional from the server to the client.

> Flow:
> Server Logic (e.g., Player Respawn) -> **Server-side Packet Creation** -> Netty Encoder -> TCP/IP Stack -> Client Netty Decoder -> **ClientTeleport.deserialize()** -> Packet Dispatcher -> Game Thread Packet Handler -> Player Entity State Update

