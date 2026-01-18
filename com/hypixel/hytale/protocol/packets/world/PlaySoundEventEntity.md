---
description: Architectural reference for PlaySoundEventEntity
---

# PlaySoundEventEntity

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class PlaySoundEventEntity implements Packet {
```

## Architecture & Concepts
The PlaySoundEventEntity class is a Data Transfer Object (DTO) that represents a specific, networked game event: playing a sound at the location of a networked entity. As an implementation of the Packet interface, its primary role is to serve as a structured, serializable container for data transmitted between the server and client.

This packet is a fundamental component of the client-server audio synchronization system. It decouples the game logic that triggers a sound from the client-side audio engine that plays it. The design prioritizes performance and low overhead. All serialization and deserialization logic operates directly on Netty ByteBuf instances, avoiding intermediate object allocations.

The class definition includes static metadata fields such as PACKET_ID, FIXED_BLOCK_SIZE, and IS_COMPRESSED. This metadata is consumed by the higher-level protocol dispatching system to efficiently parse the incoming network stream, identify the packet type, and validate its structure without needing to instantiate the object first.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Server-Side (Outgoing):** Instantiated by the server's game logic when an action requires a sound to be played for all nearby clients. For example, when an entity takes damage or performs an action.
    2. **Client-Side (Incoming):** Instantiated by the client's network protocol handler via the static deserialize method when a data block with PACKET_ID 156 is received from the server.

- **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of a single processing tick or network event callback. It is not designed to be stored, cached, or referenced long-term.

- **Destruction:** The object is eligible for garbage collection immediately after it has been serialized to the network buffer (on the server) or after its data has been consumed by the client's audio system (on the client). It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is a mutable container for its four fields: soundEventIndex, networkId, volumeModifier, and pitchModifier. While technically mutable, instances should be treated as immutable after their initial population. Modifying a packet's state after it has been queued for network transmission or received for processing is a critical anti-pattern that leads to desynchronization.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed within a single thread, typically the main game loop thread or a Netty I/O worker thread. Concurrent access from multiple threads without external synchronization will result in race conditions and undefined behavior.

## API Surface
The public API is minimal and focused entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (156). |
| serialize(ByteBuf) | void | O(1) | Writes the packet's fields into the provided buffer in little-endian format. |
| deserialize(ByteBuf, int) | PlaySoundEventEntity | O(1) | Static factory method. Reads a fixed block of 16 bytes from the buffer at an offset and constructs a new instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet's payload in bytes (16). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
A packet of this type is never managed directly. It is processed by the core networking and game event systems.

**Server-Side Sending Logic:**
```java
// Example: Server-side game logic triggering a sound
void onEntityDamaged(Entity entity, Sound sound) {
    PlaySoundEventEntity packet = new PlaySoundEventEntity(
        sound.getIndex(),
        entity.getNetworkId(),
        1.0f, // volume
        1.0f  // pitch
    );
    
    // The NetworkManager handles serialization and transmission
    // to all relevant clients.
    networkManager.sendToClientsNear(entity, packet);
}
```

**Client-Side Receiving Logic:**
```java
// Example: A client-side packet handler
public class ClientPacketHandler {
    public void handle(PlaySoundEventEntity packet) {
        // The AudioManager uses the packet data to play the sound.
        audioManager.playSoundForEntity(
            packet.networkId,
            packet.soundEventIndex,
            packet.volumeModifier,
            packet.pitchModifier
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify a packet's fields after it has been passed to the network manager for sending. The serialization may happen on a different thread or at a later time, leading to inconsistent data.
- **Object Re-use:** Do not attempt to cache and re-use packet instances. They are lightweight objects, and re-using them can lead to subtle bugs if old state is not properly cleared. Always create a new instance for each distinct event.
- **Direct Deserialization:** Do not call the deserialize method directly unless you are implementing a custom protocol handler. The core engine's packet dispatcher is responsible for reading the stream and routing bytes to the correct deserializer.

## Data Pipeline
The flow of this data object is linear and unidirectional, from server to client.

> **Server Flow:**
> Game Logic Event -> **PlaySoundEventEntity (Instantiation)** -> Network Manager Queue -> Serialization to ByteBuf -> TCP/IP Stack

> **Client Flow:**
> TCP/IP Stack -> Netty I/O Thread -> ByteBuf -> Packet Dispatcher -> **PlaySoundEventEntity (Deserialization)** -> Game Event Bus -> Audio System Consumer

