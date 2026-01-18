---
description: Architectural reference for BuilderToolOnUseInteraction
---

# BuilderToolOnUseInteraction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolOnUseInteraction implements Packet {
```

## Architecture & Concepts
The BuilderToolOnUseInteraction class is a Data Transfer Object (DTO) that represents a single, discrete player action involving a builder tool. It is a fundamental component of the client-server network protocol, acting as a structured message to communicate detailed player input from the client to the server.

This class is not a service or manager; it is a pure data container. Its primary responsibility is to serve as a data contract, defining the precise binary layout for a specific type of network packet, identified by its static PACKET_ID of 413. The class encapsulates all necessary information for the server to replicate the player's building action, including the type of interaction, target coordinates, and the player's camera orientation for raycasting.

The design prioritizes performance and network efficiency. The fixed-size structure (FIXED_BLOCK_SIZE) allows for highly optimized, zero-allocation deserialization on the server, which is critical for handling a high volume of player inputs.

## Lifecycle & Ownership
- **Creation:** An instance is created exclusively on the client-side in response to direct player input, such as clicking the mouse while a builder tool is equipped. The client's input handling system gathers the current game state (camera vectors, target block, modifier key states) and populates a new instance.
- **Scope:** The object's lifetime is extremely short and ephemeral. On the client, it exists only long enough to be serialized into a byte buffer and passed to the network transport layer. On the server, a new instance is created by the deserializer, processed by a single game tick, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. As a simple data object, it holds no native resources or references that require manual cleanup.

## Internal State & Concurrency
- **State:** The internal state is fully mutable, with all data fields exposed as public members. This design choice favors high-performance construction and serialization over encapsulation, which is a common and acceptable trade-off for internal network DTOs. The object's state represents a snapshot of player input at a specific moment in time.
- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be created, populated, and read within a single, well-defined thread context. On the client, this is the main input or game thread. On the server, it is handled by a Netty worker thread and subsequently passed to the main server tick thread. Any concurrent access would result in unpredictable behavior and data corruption.

## API Surface
The primary API consists of the serialization contract and the static deserialization factory. Direct field access is the intended method for data manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into a binary buffer for network transmission. Follows a strict, fixed-offset layout. |
| deserialize(ByteBuf, int) | static BuilderToolOnUseInteraction | O(1) | Decodes a binary buffer from a specific offset into a new object instance. This is the primary entry point on the receiving end. |
| getId() | int | O(1) | Returns the static packet identifier (413), used by the protocol dispatcher to route the packet to the correct handler. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly. It is an implementation detail of the network protocol, managed by lower-level systems. The following conceptual example illustrates how the client's input system would use it.

```java
// Conceptual client-side input handler
void onPlayerPrimaryAction() {
    // 1. Create the packet instance
    BuilderToolOnUseInteraction packet = new BuilderToolOnUseInteraction();

    // 2. Populate with current game state
    packet.type = InteractionType.Primary;
    packet.x = player.getTargetBlock().getX();
    packet.y = player.getTargetBlock().getY();
    packet.z = player.getTargetBlock().getZ();
    packet.isHoldDownInteraction = inputManager.isPrimaryActionHeld();
    
    Camera camera = player.getCamera();
    packet.raycastOriginX = camera.getPositionX();
    packet.raycastOriginY = camera.getPositionY();
    packet.raycastOriginZ = camera.getPositionZ();
    packet.raycastDirectionX = camera.getDirectionX();
    packet.raycastDirectionY = camera.getDirectionY();
    packet.raycastDirectionZ = camera.getDirectionZ();
    
    // 3. Dispatch to the network system for serialization and sending
    clientConnection.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not cache and reuse a packet instance. Each player action must generate a new instance to ensure a correct point-in-time snapshot of the game state is transmitted.
- **Server-Side Instantiation:** The server must never create an instance of this packet via its constructor. Server-side instances should only ever originate from the `deserialize` method when processing an incoming client message.
- **Asynchronous Modification:** Do not modify a packet's fields after it has been passed to the network layer for sending. The serialization process may occur on a different thread, leading to a race condition.

## Data Pipeline
This packet is a key element in the client-to-server input processing pipeline. It carries the raw data that enables server-authoritative world modification based on player actions.

> Flow:
> Client Input (Mouse Click) -> Client Input Handler -> **BuilderToolOnUseInteraction** (Instantiation) -> Client Network System (Serialization) -> TCP/IP Stream -> Server Network System (Deserialization) -> Server Packet Handler -> World Simulation (Block Placement/Removal)

