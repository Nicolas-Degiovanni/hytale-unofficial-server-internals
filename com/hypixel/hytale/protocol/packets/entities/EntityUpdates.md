---
description: Architectural reference for EntityUpdates
---

# EntityUpdates

**Package:** com.hypixel.hytale.protocol.packets.entities
**Type:** Transient

## Definition
```java
// Signature
public class EntityUpdates implements Packet {
```

## Architecture & Concepts
The EntityUpdates class is a Data Transfer Object (DTO) that represents a network packet. It is a fundamental component of the client-server state synchronization protocol, specifically for the Entity Component System (ECS). Its sole purpose is to efficiently batch and transmit entity state changes from the server to connected clients.

This packet communicates two primary types of changes:
1.  **Entity Removals:** A list of entity identifiers that should be removed from the client's world simulation. This occurs when an entity dies, is despawned, or moves out of the client's view distance.
2.  **Entity Updates:** A list of granular state changes for existing entities. Each change is encapsulated in an EntityUpdate object, which might contain new position data, health values, or other component modifications.

The binary serialization format is heavily optimized for network performance and resilience against malformed data. It employs a fixed-size header followed by a variable-data block. The header contains a **null bit field** to indicate which of the optional array fields are present, and **offsets** pointing to the start of each data array within the variable block. This design allows for fast, non-linear parsing and validation without reading the entire packet structure.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the world simulation or network layer at the end of a game tick. The server gathers all entity removals and state changes that occurred during the tick, populates a new EntityUpdates object, and queues it for serialization and broadcast to relevant clients.
    - **Client-Side:** Instantiated by the protocol's packet decoder when a raw network buffer with Packet ID 161 is received. The static *deserialize* method is called to construct the object from the byte stream.

- **Scope:** Extremely short-lived. An EntityUpdates object exists only for the duration of its creation, serialization, transmission, and deserialization. On the client, its lifetime is typically confined to a single pass of the network processing loop.

- **Destruction:** The object becomes eligible for garbage collection immediately after its contents have been applied to the client's local world state. There is no persistent state, and holding a reference to this object beyond the current network tick is a critical error, as its data will become stale.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data container with public fields. Its state is populated once during creation (either by the server logic or the client's deserializer) and is intended to be read-only thereafter.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, deserialized, and processed within a single, well-defined thread, such as a Netty IO thread or the main game thread. Passing an instance of EntityUpdates between threads requires external synchronization, which is not the intended use case. The network protocol layer is responsible for ensuring safe, single-threaded handling.

## API Surface
The primary interface is through its static factory and serialization methods, which operate on Netty ByteBufs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static EntityUpdates | O(N+M) | Constructs an EntityUpdates object by reading from a network buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N+M) | Writes the object's state into the provided network buffer according to the defined binary protocol. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N+M) | Calculates the total size of a serialized packet within a buffer without full deserialization. Essential for stream-based parsers. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N+M) | Performs a pre-flight check on a buffer to verify if it contains a structurally valid packet. Used to quickly reject invalid data. |

*N = number of removed entities, M = number of updated entities.*

## Integration Patterns

### Standard Usage
This packet is handled exclusively by the low-level networking and world synchronization layers. Application-level game logic does not interact with this class directly. Instead, it consumes the *results* of this packet's processing, typically via an event bus or by observing changes in the EntityManager.

```java
// Hypothetical consumer within the client's world synchronization logic.
// This code would execute on the main game thread after the packet is decoded.

public class ClientWorldManager {
    public void processIncomingPacket(Packet p) {
        if (p instanceof EntityUpdates packet) {
            // WARNING: This logic must be atomic for a given tick.
            if (packet.removed != null) {
                for (int entityId : packet.removed) {
                    entityManager.destroyEntity(entityId);
                }
            }
            if (packet.updates != null) {
                for (EntityUpdate update : packet.updates) {
                    entityManager.applyUpdate(update);
                }
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store an instance of EntityUpdates for later use. Its data represents a point-in-time snapshot and is immediately invalid on the next game tick.
- **Manual Instantiation on Client:** Client-side code must never create an EntityUpdates object using its constructor. It is exclusively a server-to-client message.
- **Modifying After Deserialization:** Do not modify the fields of an EntityUpdates object after it has been deserialized. Treat it as an immutable record for the duration of its processing.

## Data Pipeline
The EntityUpdates packet is a critical link in the state synchronization pipeline, carrying batched world changes from server to client.

> **Server Flow:**
> World Simulation Tick -> Generates Entity Change Set -> **EntityUpdates (Instantiation & Population)** -> Packet Serializer -> Network Buffer -> Client

> **Client Flow:**
> Network Buffer -> Packet Deserializer -> **EntityUpdates (Instantiation)** -> World Synchronization Manager -> EntityManager State Update -> Render Engine


