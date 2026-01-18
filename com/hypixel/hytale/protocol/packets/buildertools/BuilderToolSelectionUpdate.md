---
description: Architectural reference for BuilderToolSelectionUpdate
---

# BuilderToolSelectionUpdate

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class BuilderToolSelectionUpdate implements Packet {
```

## Architecture & Concepts
The BuilderToolSelectionUpdate class is a network Packet, a specialized Data Transfer Object (DTO) designed for network communication. Its sole purpose is to encapsulate the state of a player's 3D selection volume when using in-game building tools. It represents a simple, fixed-size Axis-Aligned Bounding Box (AABB) defined by two corner points (min and max).

This class sits at the lowest level of the application's protocol layer. It provides no logic beyond serialization and deserialization of its own data. The static fields like PACKET_ID, FIXED_BLOCK_SIZE, and IS_COMPRESSED serve as metadata for the higher-level Packet Registry and network dispatch systems. These systems use the metadata to identify incoming byte streams and route them to the correct deserializer, in this case, the static deserialize method of this class.

Its design prioritizes performance and predictability. By defining a fixed size and a direct mapping to a byte layout, it allows for extremely fast serialization and deserialization without reflection or complex logic, which is critical for real-time netcode.

### Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1. **Outbound:** The client-side game logic, likely a controller responsible for builder tools, instantiates this object when the player modifies their selection area. The new coordinates are passed to the constructor.
    2. **Inbound:** The network layer's packet dispatcher invokes the static deserialize method upon receiving a raw ByteBuf from the network that corresponds to this packet's ID. This factory method creates and populates a new instance from the byte stream.
- **Scope:** The object's lifetime is exceptionally short and transactional. It exists only long enough to be serialized into a buffer for sending, or from the moment it is deserialized until a corresponding PacketHandler has finished processing its data. It does not persist.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by a handler or written to the network buffer. There are no manual cleanup or resource release procedures.

## Internal State & Concurrency
- **State:** The class holds a mutable state consisting of six public integer fields representing the coordinates of the selection box. While technically mutable, instances should be treated as immutable after their initial population. Modifying a packet's state after it has been queued for network transmission is a severe anti-pattern.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms. It is designed to be created, populated, and processed within a single thread, such as a Netty event loop thread or the main game logic thread. Sharing instances across threads without external locking will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BuilderToolSelectionUpdate(int...) | constructor | O(1) | Creates a new packet with specified coordinates. |
| deserialize(ByteBuf, int) | static BuilderToolSelectionUpdate | O(1) | Constructs a new packet by reading 24 bytes from the buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the six integer fields to the buffer in little-endian format. |
| computeSize() | int | O(1) | Returns the constant size of the packet, which is 24 bytes. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (409). |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is typically created by a client-side system and sent to the server to broadcast a player's actions. The server receives it and may relay it to other clients.

```java
// Client-side: Player updates their selection
// Assume 'connection' is a handle to the network layer
BuilderToolSelectionUpdate packet = new BuilderToolSelectionUpdate(x1, y1, z1, x2, y2, z2);
connection.sendPacket(packet);

// Server-side or other clients: A PacketHandler processes the incoming data
public class BuilderToolSelectionHandler implements PacketHandler<BuilderToolSelectionUpdate> {
    public void handle(BuilderToolSelectionUpdate packet) {
        // Update the game state for the corresponding player
        Player player = getPlayerFromContext();
        player.getBuilderToolState().setSelection(packet.xMin, packet.yMin, ...);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification After Queuing:** Do not modify the public fields of a packet instance after it has been passed to the network manager for sending. The serialization may happen on a different thread at a later time, leading to a data race.
- **Instance Re-use:** Do not cache and re-use packet objects. They are lightweight and should be created for each distinct event to ensure data integrity and avoid complex state management.
- **Cross-thread Access:** Do not deserialize a packet on a network thread and process it on a game thread without a proper thread-safe handoff mechanism (e.g., a concurrent queue).

## Data Pipeline
The flow of data for this packet is linear and unidirectional for a single transmission.

> **Outbound Flow (e.g., Client to Server):**
> Player Input -> Builder Tool Controller -> **new BuilderToolSelectionUpdate()** -> NetworkManager.send() -> serialize(ByteBuf) -> Netty Channel

> **Inbound Flow (e.g., Server from Client):**
> Netty Channel -> ByteBuf -> Packet Dispatcher -> **BuilderToolSelectionUpdate.deserialize(ByteBuf)** -> PacketHandler.handle() -> Game State Update

