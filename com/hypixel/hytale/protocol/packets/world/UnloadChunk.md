---
description: Architectural reference for UnloadChunk
---

# UnloadChunk

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class UnloadChunk implements Packet {
```

## Architecture & Concepts
The UnloadChunk packet is a server-to-client network command that serves as a fundamental component of the world streaming and memory management system. It represents an explicit instruction for the client to de-allocate all resources associated with a specific vertical column of blocks, known as a chunk.

This class is a simple Data Transfer Object (DTO). Its sole purpose is to transport the coordinates of the chunk to be unloaded. It does not contain any logic for the unloading process itself. The existence of this packet is critical for preventing unbounded memory growth on the client as the player moves through the world. The server's world streaming logic tracks player proximity to chunks and issues UnloadChunk commands for regions that are no longer relevant, ensuring the client maintains a manageable memory footprint.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the server's world management system when it determines a player has moved sufficiently far from a chunk.
    - **Client-Side:** Instantiated by the protocol layer's packet deserializer when a network buffer with Packet ID 135 is received. The static factory method *deserialize* is the designated entry point.

- **Scope:** Extremely short-lived. An UnloadChunk object exists only for the duration of its processing within a single network tick. It is created, passed to a handler, and immediately becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. No manual cleanup is required. References should not be held after the packet has been processed by the relevant game system.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data container for two integer coordinates, *chunkX* and *chunkZ*. It holds no complex state, caches, or external resources. Its state is fully defined upon construction.

- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with public, mutable fields. It is designed to be created, deserialized, and processed within a single, synchronized context, such as the main client thread or a dedicated Netty event loop thread. Concurrent access from multiple threads will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type, 135. |
| serialize(ByteBuf) | void | O(1) | Encodes the chunk coordinates into a Netty byte buffer for network transmission. |
| deserialize(ByteBuf, int) | UnloadChunk | O(1) | Static factory. Decodes chunk coordinates from a byte buffer to construct a new instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes, which is always 8. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Static utility. Verifies if a buffer contains enough readable bytes to constitute a valid packet. |

## Integration Patterns

### Standard Usage
The packet is decoded by the network layer and immediately dispatched to a higher-level system, such as a WorldManager, which is responsible for acting upon the command.

```java
// Executed within a network event handler
// packetBuffer is a ByteBuf received from the server
UnloadChunk packet = UnloadChunk.deserialize(packetBuffer, 0);

// Dispatch the command to the world system
WorldManager worldManager = context.getService(WorldManager.class);
worldManager.queueChunkUnload(packet.chunkX, packet.chunkZ);
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** Do not use *new UnloadChunk()* on the client for any game logic. These packets are commands *from* the server. Creating them on the client has no effect.
- **State Mutation:** Do not modify the *chunkX* or *chunkZ* fields after the packet has been deserialized. The object should be treated as immutable upon receipt to prevent unloading the wrong chunk.
- **Reference Caching:** Do not store references to UnloadChunk packets in long-lived collections. They are transient and should be processed and discarded immediately to avoid memory leaks.

## Data Pipeline
The flow of this data object is unidirectional from server to client.

> **Server Flow:**
> World Streaming Logic -> **UnloadChunk (new)** -> Protocol Encoder -> TCP/IP Stack

> **Client Flow:**
> TCP/IP Stack -> Netty ByteBuf -> Protocol Decoder -> **UnloadChunk (deserialize)** -> World Event Bus -> WorldManager -> Chunk De-allocation
---

