---
description: Architectural reference for MountNPC
---

# MountNPC

**Package:** com.hypixel.hytale.protocol.packets.interaction
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class MountNPC implements Packet {
```

## Architecture & Concepts
The MountNPC class is a Data Transfer Object (DTO) that represents a specific, fixed-size network message. It is a concrete implementation of the Packet interface, which standardizes the contract for all network communication payloads within the Hytale protocol layer.

Its primary role is to encapsulate the data required for a client to request mounting a specific non-player character (NPC) entity from the server. The class provides the serialization and deserialization logic necessary to convert its state to and from the raw byte stream managed by the Netty framework.

The static fields, such as PACKET_ID and FIXED_BLOCK_SIZE, serve as critical metadata for the protocol's central packet dispatcher. This dispatcher uses the PACKET_ID to map an incoming byte stream to the correct packet type and uses the size constants to perform efficient, low-level buffer validation and processing without needing to instantiate the object first.

## Lifecycle & Ownership
- **Creation:** An instance of MountNPC is created under two distinct circumstances:
    1. **Sending Peer:** The game client's logic instantiates this object when a player initiates an action to mount an NPC. The relevant entity ID and anchor coordinates are passed to the constructor.
    2. **Receiving Peer:** The network protocol layer instantiates this object via the static deserialize factory method. This occurs after the packet's ID has been read from the incoming ByteBuf and identified as 293.

- **Scope:** The object's lifetime is exceptionally brief and transactional. It exists only for the duration of a single network operation. On the sending side, it is typically discarded immediately after its serialize method is called. On the receiving side, it is discarded after the corresponding packet handler has processed its data.

- **Destruction:** The object is managed by the Java Garbage Collector and becomes eligible for collection as soon as it is no longer referenced by the network stack or the game logic handler. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency
- **State:** The MountNPC object is a simple, mutable data container. Its state consists of four public fields: anchorX, anchorY, anchorZ, and entityId. It does not maintain any internal cache or complex state machine. Its purpose is to be a transparent carrier of data.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, serialized, and processed within a single thread context, such as a Netty I/O worker thread or the main game logic thread.

    **Warning:** Sharing a MountNPC instance across multiple threads without explicit, external synchronization will lead to race conditions and unpredictable behavior. Received packets should be considered immutable by application logic to prevent state corruption.

## API Surface
The public API is designed for interaction with the protocol framework, not for general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MountNPC(float, float, float, int) | constructor | O(1) | Constructs a new packet for outbound transmission. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a Netty byte buffer using little-endian format. |
| deserialize(ByteBuf, int) | static MountNPC | O(1) | Constructs a new packet by reading a fixed block of 16 bytes from a buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid packet. |
| getId() | int | O(1) | Returns the static packet identifier (293). Used by the protocol dispatcher. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes (16). |

## Integration Patterns

### Standard Usage
This packet is processed by a server-side handler that receives it from the network layer. The handler extracts the entity ID and uses it to update the game state.

```java
// Example of a server-side packet handler
void handleMountRequest(MountNPC packet) {
    // Retrieve the world or entity manager from the current context
    World world = server.getWorld();
    
    // Find the player entity who sent the request and the target NPC
    PlayerEntity player = getPlayerFromContext();
    NPCEntity targetNpc = world.findEntityById(packet.entityId, NPCEntity.class);

    if (targetNpc != null) {
        // Delegate to the game logic system to handle the mount action
        player.getMountingController().mount(targetNpc, packet.anchorX, packet.anchorY, packet.anchorZ);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send a packet instance that has been received from the network. Always create a new instance for outbound messages.
- **Cross-Thread Access:** Do not pass a received packet instance to another thread. Instead, extract its primitive data (entityId, coordinates) and pass those values. This prevents concurrency issues.
- **Manual Serialization:** Do not attempt to read or write the fields manually to a buffer. Always use the provided serialize and deserialize methods to ensure correctness with the protocol specification, especially regarding byte order (little-endian).

## Data Pipeline
The MountNPC packet is a key component in the client-to-server interaction data flow.

> **Flow (Client Request -> Server):**
> Player Input (e.g., "E" key press) -> Client-side Interaction Logic -> **new MountNPC(x, y, z, id)** -> Packet.serialize() -> Netty ByteBuf -> Server Network Layer -> Packet Decoder -> **MountNPC.deserialize()** -> Server-side Packet Handler -> World State Update

