---
description: Architectural reference for SetChunkHeightmap
---

# SetChunkHeightmap

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class SetChunkHeightmap implements Packet {
```

## Architecture & Concepts
The SetChunkHeightmap class is a Data Transfer Object (DTO) that represents a single, specific network message within the Hytale world streaming protocol. It is not a service or a manager; it is a passive data structure whose sole purpose is to encapsulate the heightmap information for a vertical chunk column.

This packet is fundamental to the client's ability to render terrain and perform physics calculations before the full chunk data has arrived. The heightmap provides a low-detail vertical profile of the terrain, defining the highest solid block at each X,Z coordinate within a chunk. This allows the engine to construct a preliminary, low-fidelity version of the world for distant rendering, occlusion culling, and basic collision.

It acts as a direct, language-specific representation of a wire-format message. Its structure is rigidly defined by the network protocol, with methods for serialization and deserialization that convert the object's state to and from a Netty ByteBuf.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated via its constructor (`new SetChunkHeightmap(...)`) by the world generation or world streaming subsystem when a chunk's heightmap needs to be sent to a client.
    - **Client-Side:** Instantiated exclusively by the network protocol layer. The static `deserialize` method is invoked by a packet decoder when a message with PACKET_ID 132 is received from the server.

- **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of its processing within a single network tick or game loop iteration.

- **Destruction:** The object is managed by the Java Garbage Collector. Once the client's WorldManager consumes the heightmap data and updates its internal world state, all references to the SetChunkHeightmap instance are dropped, making it eligible for garbage collection. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The state is mutable and consists of the chunk's world coordinates (x, z) and the heightmap data itself as a byte array. The `heightmap` field is nullable, which is handled by a bitfield during serialization to optimize network traffic for empty or un-generated heightmaps.

- **Thread Safety:** This class is **not thread-safe**. As a simple DTO, it possesses no internal locking mechanisms. It is designed to be created, processed, and discarded by a single thread, typically a Netty I/O worker thread or a main game logic thread.

    **WARNING:** Sharing an instance of SetChunkHeightmap across multiple threads without external synchronization will lead to undefined behavior and memory consistency errors.

## API Surface
The public API is dominated by static methods for protocol handling and instance methods for serialization, reflecting its role as a network packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static SetChunkHeightmap | O(N) | Constructs a new instance by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the size of the heightmap. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(1) | Calculates the exact byte size the serialized packet will occupy. Does not account for the heightmap array itself. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a valid packet structure without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (132). |

## Integration Patterns

### Standard Usage
The packet is handled by a client-side network listener. The listener extracts the data and forwards it to the appropriate world subsystem, which then applies the heightmap to the in-memory representation of the corresponding chunk.

```java
// In a client-side packet handler
void handlePacket(SetChunkHeightmap packet) {
    // Retrieve the world management service
    WorldManager worldManager = context.getService(WorldManager.class);

    // Apply the heightmap data to the world state
    worldManager.updateChunkHeightmap(packet.x, packet.z, packet.heightmap);

    // The 'packet' object is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** A client should never create this packet using `new SetChunkHeightmap()`. It is a server-to-client message. Doing so would have no effect on the game state.
- **Retaining References:** Do not store instances of this packet in long-lived collections or as member variables in services. Its data should be copied into the canonical world data structures immediately, and the packet object itself should be discarded.
- **Ignoring Nullability:** The `heightmap` field can be null. Code must always check for null before attempting to access the array to prevent NullPointerException. The serialization logic handles this with a bitmask.

## Data Pipeline
The flow of data represented by this class is unidirectional from server to client.

> **Server Flow:**
> World State -> Chunk Serializer -> **new SetChunkHeightmap(x, z, data)** -> serialize() -> Netty Encoder -> Network Socket

> **Client Flow:**
> Network Socket -> Netty Decoder -> **SetChunkHeightmap.deserialize()** -> Packet Handler -> WorldManager -> Chunk Data Update

