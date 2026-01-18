---
description: Architectural reference for SetChunkTintmap
---

# SetChunkTintmap

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class SetChunkTintmap implements Packet {
```

## Architecture & Concepts
The SetChunkTintmap class is a Data Transfer Object (DTO) that represents a single, atomic network message. It is not a service or manager; it is a pure data container used within the Hytale world-streaming protocol.

Its primary function is to transmit biome-specific or environmentally-influenced color data for a specific vertical chunk of the game world, identified by its X and Z coordinates. This data, the *tintmap*, is used by the client's rendering engine to apply dynamic coloring to blocks, foliage, and water, creating visually diverse and cohesive environments.

This class embodies a common pattern in high-performance networking: co-locating serialization and deserialization logic with the data structure itself. The static `deserialize` and instance `serialize` methods act as the marshalling boundary between the raw byte stream managed by Netty and the structured, in-memory representation of the game protocol.

**WARNING:** This object represents a point-in-time snapshot of server state. It is not a live view and should be treated as immutable after deserialization.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the world simulation or streaming logic when a chunk's tintmap needs to be sent or updated for a connected client.
    - **Client-Side:** Instantiated exclusively by the protocol decoding layer. The static `deserialize` method serves as the factory, constructing the object from an incoming Netty ByteBuf when a packet with ID 133 is detected.
- **Scope:** Extremely short-lived. A SetChunkTintmap object exists only for the brief moment between network deserialization and the application of its data to the client's world state.
- **Destruction:** The object becomes eligible for garbage collection immediately after its payload has been processed by a world data handler. Holding long-term references to packet objects is a critical memory leak anti-pattern.

## Internal State & Concurrency
- **State:** Mutable. The object's fields are public and are populated during the deserialization process. The internal `tintmap` byte array is a mutable structure.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded within the confines of a single network thread (e.g., a Netty I/O worker). Passing an instance of this packet across thread boundaries without proper synchronization or defensive copying of its internal array is a severe concurrency bug. The intended pattern is to process it on the network thread and dispatch an immutable value or a new command to the main game thread.

## API Surface
The public API is dominated by static utility methods for interacting with raw byte buffers, reflecting its role as a protocol definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static SetChunkTintmap | O(N) | Constructs a new packet by reading from a buffer. N is the size of the tintmap. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the packet's state into the provided buffer. N is the size of the tintmap. Throws ProtocolException if data exceeds limits. |
| computeSize() | int | O(1) | Calculates the number of bytes this packet will occupy when serialized. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a cheap, non-allocating check to see if a buffer likely contains a valid packet of this type. |
| getId() | int | O(1) | Returns the static network identifier for this packet type, which is 133. |

## Integration Patterns

### Standard Usage
The canonical use case involves a network handler that decodes the packet and immediately dispatches its data to the appropriate game system.

```java
// In a client-side network handler...
// buf is an incoming ByteBuf from Netty
SetChunkTintmap packet = SetChunkTintmap.deserialize(buf, offset);

// Immediately pass the data to the world system.
// Do not hold a reference to the 'packet' object itself.
WorldManager world = context.getService(WorldManager.class);
world.updateChunkTintmap(packet.x, packet.z, packet.tintmap);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** A client should never call `new SetChunkTintmap()`. These packets are exclusively received from the server.
- **Long-Term Storage:** Storing instances of SetChunkTintmap in a collection or as a member variable is a memory leak. Extract the data you need and discard the packet object.
- **State Mutation:** Modifying the packet's fields after deserialization can lead to an inconsistent view of the world state. Treat it as a read-only structure once created.
- **Cross-Thread Sharing:** Never pass the packet object directly from the network thread to the main game thread. This will lead to race conditions. Instead, dispatch an event or command containing copies of the necessary data.

## Data Pipeline
The SetChunkTintmap packet is a critical link in the world data streaming pipeline, responsible for visual fidelity.

> Flow:
> Server World Engine -> **SetChunkTintmap (Serialization)** -> TCP Stream -> Client Network Decoder -> **SetChunkTintmap (Deserialization)** -> World Update Handler -> Chunk Render Data Update

