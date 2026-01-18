---
description: Architectural reference for BuilderToolEntityAction
---

# BuilderToolEntityAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolEntityAction implements Packet {
```

## Architecture & Concepts
The BuilderToolEntityAction class is a network packet definition, not a service or manager. It functions as a simple, fixed-size Data Transfer Object (DTO) that represents a discrete command sent from a client to the server when using in-game builder tools. Its specific purpose is to communicate a single action, such as removal, to be performed on a specific entity.

This packet is part of the low-level protocol layer. Its design prioritizes performance and minimal overhead for high-frequency operations. Key architectural indicators include:
- **Fixed Size:** The packet has a constant wire size of 5 bytes, defined by FIXED_BLOCK_SIZE. This allows for extremely fast deserialization and validation without parsing variable-length fields.
- **No Compression:** The IS_COMPRESSED flag is false, indicating that the system avoids the overhead of compression/decompression for this small, common packet.
- **Static Deserializer:** The presence of a static `deserialize` factory method is a common pattern in the Hytale protocol layer, allowing the network pipeline to instantiate packets without reflection.

This class is a fundamental building block for server-authoritative game logic. The client sends a command, and the server validates and executes it, ensuring a consistent world state.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Client):** Instantiated by game logic in response to player input. For example, when a player uses a "remove" tool on an entity, the input handling system creates a new BuilderToolEntityAction, populates the entityId and action, and queues it for network transmission.
    - **Receiving Peer (Server):** Instantiated by the network protocol decoder. The static `deserialize` method is called with a raw Netty ByteBuf, constructing a new packet instance from the incoming data stream.
- **Scope:** Extremely short-lived and transient. An instance of this packet exists only for the brief period between its creation and its consumption by a handler. It is a message, not a persistent state container.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed by the target system (e.g., a server-side packet handler). There is no manual memory management or destruction logic associated with this class.

## Internal State & Concurrency
- **State:** The class holds a small, mutable state consisting of an `entityId` and an `action`. The fields are public for high-performance access during serialization and handling, a common trade-off for DTOs in performance-critical code.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, deserialized, and handled within a single thread, typically a Netty I/O worker thread.
    - **WARNING:** Modifying a BuilderToolEntityAction instance from multiple threads before serialization will result in a race condition and undefined network data. All state must be finalized on the originating thread before the object is passed to the network pipeline.

## API Surface
The public contract is focused entirely on serialization, deserialization, and data validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided network buffer. |
| deserialize(ByteBuf, int) | static BuilderToolEntityAction | O(1) | Static factory method. Decodes a new instance from a network buffer at a given offset. |
| computeSize() | int | O(1) | Returns the constant size of the packet on the wire (5 bytes). |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough bytes to read the packet, preventing underflow exceptions. |

## Integration Patterns

### Standard Usage
This packet is never used directly by high-level game feature developers. It is processed by the server's central packet handling system, which then dispatches the data to the appropriate game logic.

```java
// Example of a server-side packet handler consuming the packet
public class BuilderToolPacketHandler implements PacketHandler<BuilderToolEntityAction> {

    @Override
    public void handle(PlayerConnection connection, BuilderToolEntityAction packet) {
        World world = connection.getPlayer().getWorld();
        Entity targetEntity = world.findEntityById(packet.entityId);

        // Validate permissions and game rules before executing
        if (targetEntity != null && connection.hasBuildPermission(targetEntity.getPosition())) {
            if (packet.action == EntityToolAction.Remove) {
                world.removeEntity(targetEntity);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not cache or reuse a packet instance. Each action must generate a new instance to represent its unique, point-in-time state. Holding a reference to a packet after it has been handled can lead to processing stale data.
- **Manual Deserialization:** On the receiving end, always use the static `deserialize` method. Do not use `new BuilderToolEntityAction()` and manually read from the ByteBuf. This would violate the serialization contract and make the code brittle to future format changes.
- **Client-Side Execution:** Do not have the client act on this packet directly. The client should send the packet and wait for the server to send a world update. This maintains the server-authoritative model.

## Data Pipeline
The flow of this data object is linear and unidirectional for a single command.

> Flow:
> Client Input System -> Game Logic Instantiation -> **BuilderToolEntityAction** -> Protocol Serializer -> Network Channel -> Protocol Deserializer -> **BuilderToolEntityAction** (New Instance) -> Server Packet Handler -> World State Change

