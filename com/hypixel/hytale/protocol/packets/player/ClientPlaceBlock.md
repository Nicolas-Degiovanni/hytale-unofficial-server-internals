---
description: Architectural reference for ClientPlaceBlock
---

# ClientPlaceBlock

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Packet Model / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ClientPlaceBlock implements Packet {
```

## Architecture & Concepts
The ClientPlaceBlock class is a network protocol Data Transfer Object. It is not a service or manager, but a structured data container representing a specific, client-initiated event: a player placing a block in the world. This class forms a critical part of the client-server contract, defining the precise binary layout for the "place block" action.

Its primary architectural role is to encapsulate user input into a serializable format that the server can unambiguously interpret. The design prioritizes network performance and parsing efficiency over encapsulation. This is evident in its public fields and fixed-size structure.

A key design feature is its fixed 20-byte size, which simplifies buffer management and eliminates the need for dynamic size prefixes on the wire. This is achieved by using a bitmask, `nullBits`, to indicate the presence of nullable fields like `position` and `rotation`, while still allocating a constant space for them in the payload. This trades a small amount of potential network bandwidth for a significant gain in parsing speed and simplicity.

## Lifecycle & Ownership
- **Creation:** An instance of ClientPlaceBlock is created on the game client whenever a player executes the action to place a block. The client-side game logic populates the instance with the target position, rotation, and the ID of the block being placed.

- **Scope:** This object is extremely transient and has a very short lifecycle. It exists only for the brief period required to be serialized into a network buffer on the client, and subsequently, to be deserialized from a buffer on the server for processing.

- **Destruction:** The object is immediately eligible for garbage collection after its data has been processed. On the client, this occurs after serialization. On the server, this occurs after the game logic has consumed the data from the deserialized object to update the world state. It is never cached or retained long-term.

## Internal State & Concurrency
- **State:** The object's state is entirely mutable, with public fields for direct access. This is a deliberate design choice for a high-performance DTO, minimizing the overhead of getter/setter method calls during the critical serialization and deserialization paths.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, populated, and read by a single thread within the context of either the game loop or a dedicated network thread.

> **Warning:** Sharing a ClientPlaceBlock instance across multiple threads is a severe anti-pattern and will result in data corruption and unpredictable network payloads. Treat instances as thread-local data.

## API Surface
The public contract is focused exclusively on serialization, deserialization, and metadata for the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided Netty ByteBuf according to the fixed 20-byte layout. |
| deserialize(ByteBuf, int) | static ClientPlaceBlock | O(1) | A static factory method that decodes a 20-byte segment of a ByteBuf into a new ClientPlaceBlock instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload, which is always 20 bytes. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-deserialization check to ensure the buffer contains enough readable bytes to prevent a buffer underflow. |
| getId() | int | O(1) | Returns the unique packet identifier, 117, used by the protocol dispatcher to route the raw bytes to the correct deserializer. |

## Integration Patterns

### Standard Usage
This object is never directly instantiated by most game logic. Instead, it is created by low-level input handlers on the client and consumed by packet processors on the server. A typical server-side handler would receive the object after it has been deserialized by the network layer.

```java
// Hypothetical server-side packet handler
public class PlayerActionHandler {
    public void handlePlaceBlock(Player player, ClientPlaceBlock packet) {
        if (packet.position == null) {
            // Invalid packet, player might be desynced or cheating
            return;
        }

        World world = player.getWorld();
        // The server validates the action before modifying the world state
        if (world.canPlaceBlockAt(player, packet.position)) {
            world.setBlock(packet.position, packet.placedBlockId, packet.rotation);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-serialize the same ClientPlaceBlock instance. Each distinct player action must result in a new instance to ensure data integrity.
- **Manual Serialization:** Never attempt to read or write the fields to a buffer manually. The `serialize` and `deserialize` methods correctly handle the `nullBits` bitmask, and bypassing them will lead to protocol errors.
- **Long-Term Storage:** This object represents a transient event. Do not store it in caches, databases, or other long-lived collections. If the action needs to be recorded, extract its data into a dedicated persistence model.

## Data Pipeline
The ClientPlaceBlock object is a payload that travels through the core networking pipeline. It is data, not logic.

> Flow:
> Client Input -> Client Game Logic -> **ClientPlaceBlock (Instance Created)** -> Network Layer (serialize) -> ByteBuf -> Server Network Layer (receive) -> Packet Dispatcher -> **ClientPlaceBlock (Instance Deserialized)** -> Server Game Logic -> World State Update

