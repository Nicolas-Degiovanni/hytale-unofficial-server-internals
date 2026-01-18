---
description: Architectural reference for UpdateBlockDamage
---

# UpdateBlockDamage

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class UpdateBlockDamage implements Packet {
```

## Architecture & Concepts
The UpdateBlockDamage class is a network packet definition, a fundamental component of the Hytale protocol layer. It serves as a structured data container, or Data Transfer Object (DTO), for communicating real-time changes to a block's integrity from the server to the client.

This class is not a service or a manager; it is a pure data structure. Its primary role is to be serialized into a byte stream for network transmission and deserialized back into an object on the receiving end. The design emphasizes performance and minimal overhead, evident in its fixed size, manual serialization logic, and bitmasking for nullable fields. It represents a single, atomic event within the game world: a change in the damage level of a specific block.

## Lifecycle & Ownership
- **Creation:** On the server, an instance is created by the game logic (e.g., the World or PlayerInteraction system) when a player deals damage to a block. On the client, an instance is created exclusively by the protocol's deserialization pipeline when a network buffer with packet ID 144 is processed.
- **Scope:** Extremely short-lived. An UpdateBlockDamage object exists only for the brief moment it takes to be processed. On the server, it is created, serialized, and immediately becomes eligible for garbage collection. On the client, it is deserialized, passed to one or more event handlers, and then becomes eligible for garbage collection, typically within a single network tick.
- **Destruction:** Memory is reclaimed by the Java Garbage Collector. There is no manual destruction logic. Holding a reference to a packet after its processing cycle is complete is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** Mutable. The class contains public fields for blockPosition, damage, and delta. This design facilitates rapid population during deserialization or before serialization. It holds no internal caches or complex state beyond the data it represents.
- **Thread Safety:** This class is **not thread-safe**. Packet objects are designed to be processed by a single thread, typically a Netty event loop thread. Accessing or modifying a packet instance from multiple threads without explicit, external synchronization will result in data corruption and unpredictable client-side behavior. All processing should be dispatched to the main game thread if necessary.

## API Surface
The public API is centered around the serialization contract required by the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (144). |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into a fixed-size binary format. |
| deserialize(ByteBuf, int) | UpdateBlockDamage | O(1) | Static factory method to construct an object from a network buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized packet in bytes (21). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a low-level check to ensure the buffer is large enough to contain the packet. |

## Integration Patterns

### Standard Usage
This packet is consumed by client-side systems that manage world state and rendering. A handler listens for this specific packet type, extracts its data, and applies the update to the appropriate game systems.

```java
// Hypothetical client-side packet handler
public class ClientWorldManager implements PacketListener {

    public void onPacketReceived(UpdateBlockDamage packet) {
        // Ensure processing happens on the main game thread
        GameThread.dispatch(() -> {
            BlockPosition pos = packet.blockPosition;
            if (pos != null) {
                // Update the visual effect for block damage
                worldRenderer.setBlockDamageEffect(pos, packet.damage);

                // Update internal game state if necessary
                localWorldState.updateBlockIntegrity(pos, packet.damage);
            }
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify a received packet. Treat it as an immutable record upon receipt. Modifying its fields can cause inconsistent state if multiple systems are listening for the same packet instance.
- **Long-Term Storage:** Do not store packet instances in collections or as member variables. They are transient and should be processed immediately. Copy the data into your own long-lived data structures if persistence is required.
- **Cross-Thread Processing:** Never process a packet on a network I/O thread if the logic involves accessing shared game state. Always dispatch the packet or its data to a synchronized game loop or main thread.

## Data Pipeline
The flow of this data is unidirectional from server to client, triggered by a player action.

> **Flow:**
> Player Action (Server) -> World Simulation -> **UpdateBlockDamage (created)** -> Packet Serializer -> Netty Channel -> Network -> Netty Channel (Client) -> Packet Deserializer -> **UpdateBlockDamage (re-created)** -> Packet Handler -> World Renderer -> Visual Update (Crack effect on block)

