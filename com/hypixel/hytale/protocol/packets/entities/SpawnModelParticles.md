---
description: Architectural reference for SpawnModelParticles
---

# SpawnModelParticles

**Package:** com.hypixel.hytale.protocol.packets.entities
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class SpawnModelParticles implements Packet {
```

## Architecture & Concepts
The SpawnModelParticles class is a Data Transfer Object (DTO) that represents a single, specific message within the Hytale network protocol. It is not a service or a manager; it is a pure data container with no inherent logic beyond serialization and deserialization.

Its primary role is to carry the necessary information from the server to the client to trigger the rendering of one or more particle effects attached to a specific entity model. This class acts as a structured representation of a raw byte stream, providing a type-safe way for the game engine to interact with network data. It exists at the boundary between the low-level network transport layer (Netty) and the higher-level game simulation and rendering systems.

The design is optimized for performance, with static methods for deserialization and validation that operate directly on Netty ByteBuf objects. This avoids intermediate object creation and allows for efficient, zero-copy-style reads where possible.

## Lifecycle & Ownership
-   **Creation:** An instance of SpawnModelParticles is created under two circumstances:
    1.  **Inbound:** The network protocol decoder instantiates it by calling the static deserialize method when a packet with ID 165 is received from the server.
    2.  **Outbound:** The server-side game logic instantiates it via its constructor to prepare a message for sending to clients.

-   **Scope:** This object is extremely short-lived and transaction-scoped. It is designed to exist only for the duration of a single network event processing cycle. Once the data it contains has been consumed by a game system (e.g., the ParticleManager), the object should be considered stale.

-   **Destruction:** The object is managed by the Java Garbage Collector. As it is intended for transient use, it becomes eligible for collection as soon as all references to it are dropped, which typically occurs upon completion of the event handler that processed it.

**Warning:** Holding a long-term reference to a packet object is a design error and may constitute a memory leak. These objects represent a point-in-time state and should not be cached.

## Internal State & Concurrency
-   **State:** The class is fully mutable. Its public fields, entityId and modelParticles, can be directly modified after instantiation. This design choice prioritizes performance and ease of use within the single-threaded context of packet processing over encapsulation.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. Network packets are processed sequentially within a dedicated network thread (e.g., a Netty EventLoop) and are typically handed off to the main game thread via a queue. All reads and writes to a SpawnModelParticles instance must be confined to a single thread to prevent data corruption and race conditions.

## API Surface
The public API is focused entirely on protocol-level operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | SpawnModelParticles | O(N) | **Static Factory.** Deserializes a packet from a raw ByteBuf. N is the number of particles. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into a ByteBuf for network transmission. N is the number of particles. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. Used for pre-allocating buffers. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **Static.** Performs a read-only check of the buffer to ensure data is well-formed before attempting a full deserialization. |
| clone() | SpawnModelParticles | O(N) | Creates a deep copy of the packet and its contained particle data. |

## Integration Patterns

### Standard Usage
The object is deserialized by the network layer and immediately dispatched to a handler or event bus, which consumes its data to update the game state.

```java
// In a network pipeline handler...
// byteBuf contains the raw data for a SpawnModelParticles packet.

SpawnModelParticles packet = SpawnModelParticles.deserialize(byteBuf, 0);

// Dispatch to the main game thread for processing
GameEventBus.post(new ParticleSpawnEvent(packet.entityId, packet.modelParticles));
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not modify and re-serialize a packet instance that was received from the network. Always create a new instance for outbound messages.
-   **Cross-Thread Access:** Never pass a reference to this object to another thread without a deep copy or proper synchronization, which is against its intended design. The receiving system should extract the data, not the packet object itself.
-   **Long-Term Storage:** Do not store SpawnModelParticles instances in collections or as member variables of long-lived services. Extract the primitive data you need and discard the packet.

## Data Pipeline
The SpawnModelParticles packet is a critical component in the visual effects data pipeline. It translates a server-side game event into a client-side visual representation.

> Flow:
> Server Game Event -> **new SpawnModelParticles()** -> Server-side Serialization -> Network Transport (TCP/UDP) -> Client-side Deserialization -> **SpawnModelParticles Instance** -> Game Event Bus -> Particle Rendering System -> GPU Command Buffer -> Particles Rendered on Screen

