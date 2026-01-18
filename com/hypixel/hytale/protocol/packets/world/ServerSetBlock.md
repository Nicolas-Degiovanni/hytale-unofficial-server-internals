---
description: Architectural reference for ServerSetBlock
---

# ServerSetBlock

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class ServerSetBlock implements Packet {
```

## Architecture & Concepts
The ServerSetBlock class is a Data Transfer Object (DTO) that represents a single, atomic world modification command sent from the server to the client. It is a fundamental component of the world state synchronization protocol.

This packet is unidirectional, flowing exclusively from the server to the client. Its purpose is to instruct the client's world representation to update a specific block at a given coordinate to a new state, including its type (blockId) and orientation (rotation). It is a highly optimized, fixed-size packet designed for high-frequency transmission, critical for real-time environmental changes like player building, explosions, or world generation streams.

It operates at the core of the network protocol layer, acting as a serialized message that is decoded by the client's network thread and dispatched to the appropriate game logic handler.

### Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the server's World or Chunk management system whenever a block's state is authoritatively changed.
    - **Client-Side:** Instantiated by the packet deserialization pipeline when an incoming network buffer with Packet ID 140 is processed. It is never created manually in client game logic.
- **Scope:** Extremely short-lived. An instance of ServerSetBlock exists only for the brief period between its deserialization from the network buffer and the completion of its processing by a packet handler. It is a message, not a persistent entity.
- **Destruction:** The object is eligible for garbage collection immediately after the client-side packet handler has consumed its data. There is no manual memory management or pooling mechanism apparent from its definition.

## Internal State & Concurrency
- **State:** The object's state is entirely mutable, consisting of public fields for block coordinates, ID, and metadata. It is a simple data container with no internal logic for state management. Once deserialized on the client, its state should be considered final and read-only.

- **Thread Safety:** **Not thread-safe.** This class is a plain data object with no synchronization mechanisms. It is designed to be created, processed, and discarded within a single thread, typically a Netty network event loop or the main client game thread.

    **WARNING:** Sharing an instance of ServerSetBlock across threads without external synchronization will lead to race conditions and unpredictable client behavior. It must be processed completely by the receiving thread or have its data copied to a thread-safe structure before being handed off.

## API Surface
The public contract is divided between static factory methods for creation from a network buffer and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerSetBlock() | constructor | O(1) | Creates an uninitialized packet. Primarily for internal deserializer use. |
| ServerSetBlock(int, int, int, int, short, byte) | constructor | O(1) | Creates a fully-initialized packet. Primarily for server-side use. |
| deserialize(ByteBuf, int) | static ServerSetBlock | O(1) | Constructs and populates a new instance from a Netty ByteBuf at a given offset. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains enough data for a valid packet. |
| serialize(ByteBuf) | void | O(1) | Writes the packet's fields into the provided ByteBuf using little-endian byte order. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes, which is always 19. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (140). |

## Integration Patterns

### Standard Usage
The packet is handled by a registered listener, which extracts the data and forwards it to the client's world system to apply the change. This pattern ensures separation of network logic from game logic.

```java
// In a client-side PacketHandler implementation
public void handle(ServerSetBlock packet) {
    // Retrieve the client's world representation from the game context
    ClientWorld world = game.getWorld();

    // Apply the state change authoritatively
    // The ClientWorld is responsible for updating the chunk, mesh, and physics data
    world.setBlockState(packet.x, packet.y, packet.z, packet.blockId, packet.rotation);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Do not use `new ServerSetBlock()` on the client side for game logic. These packets represent server commands and should only originate from the network deserializer.
- **State Modification:** Do not modify the fields of a ServerSetBlock packet after it has been received. Treat it as an immutable record once it leaves the deserializer. Modifying it can cause downstream systems to process corrupted or incorrect data.
- **Instance Caching:** Do not cache or reuse ServerSetBlock instances. Their lifecycle is meant to be transient. Caching can lead to complex state management issues and bugs where old data is accidentally processed.

## Data Pipeline
The flow of data for a block update is linear and travels from the server's core logic to the client's rendering engine.

> Flow:
> Server World Change -> **ServerSetBlock (Server-Side Creation)** -> Network Serializer -> TCP/IP Stack -> Client Network Listener -> Packet Deserializer -> **ServerSetBlock (Client-Side Creation)** -> Packet Handler -> ClientWorld System -> Chunk Mesh Rebuild -> Render Engine

