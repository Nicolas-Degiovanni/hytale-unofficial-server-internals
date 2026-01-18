---
description: Architectural reference for UpdateTimeSettings
---

# UpdateTimeSettings

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient Data Object (DTO)

## Definition
```java
// Signature
public class UpdateTimeSettings implements Packet {
```

## Architecture & Concepts
The UpdateTimeSettings class is a network packet data structure, not a service or manager. It serves as a Data Transfer Object (DTO) within the Hytale network protocol, specifically for synchronizing the world's temporal state from the server to the client.

This packet is the single source of truth for the client's understanding of the day-night cycle, moon phases, and whether time is currently advancing. The server dictates these parameters, and clients are expected to adjust their local world simulation and rendering accordingly upon receipt. It is a fundamental component of the world state synchronization mechanism, ensuring a consistent experience for all players on a server.

Its design is optimized for network performance: it has a fixed, small size and contains no variable-length fields, allowing for highly efficient serialization and deserialization without complex parsing logic.

## Lifecycle & Ownership
- **Creation:**
    - **Server-side:** Instantiated by the server's world logic when the time settings need to be communicated. This typically occurs when a player first joins the world or when a game administrator modifies the time settings via a command.
    - **Client-side:** Instantiated exclusively by the network protocol decoder when a raw byte buffer with Packet ID 145 is received from the server. The static `deserialize` method is the designated factory on the client.

- **Scope:** The object's lifetime is extremely brief and transactional. It exists only long enough to be serialized into a byte buffer on the server or to have its data read by a packet handler on the client.

- **Destruction:** It is a short-lived object with no persistent references. Once its data has been transferred to or from the network buffer and processed by the relevant handler, it becomes immediately eligible for garbage collection.

## Internal State & Concurrency
- **State:** The class is a mutable data container with public fields. Its state represents a snapshot of the server's time configuration at a specific moment. While technically mutable, it is treated as immutable after creation (on the server) or deserialization (on the client).

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, populated, and processed by a single thread within the server's game loop or the client's network thread.

    **WARNING:** Accessing or modifying an instance of UpdateTimeSettings from multiple threads will lead to race conditions and undefined behavior. The network layer guarantees that packet processing is serialized, mitigating this risk in standard usage.

## API Surface
The primary API is for interaction with the network protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | UpdateTimeSettings | O(1) | **[Client]** Constructs a new packet by reading 10 bytes from the provided ByteBuf. |
| serialize(buf) | void | O(1) | **[Server]** Writes the packet's state into the provided ByteBuf using little-endian format. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes, which is always 10. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Checks if the buffer contains enough readable bytes to deserialize the packet. |

## Integration Patterns

### Standard Usage
This packet is handled internally by the network engine. A developer would typically interact with the *result* of this packet via a higher-level event or service, not the packet object itself.

On the client, a packet handler would receive the deserialized object and apply its state to the world.

```java
// Hypothetical client-side packet handler
public void handle(UpdateTimeSettings packet) {
    WorldTimeManager timeManager = context.getService(WorldTimeManager.class);
    
    // Transfer data from the transient packet to a persistent service
    timeManager.setDayDuration(packet.daytimeDurationSeconds);
    timeManager.setNightDuration(packet.nighttimeDurationSeconds);
    timeManager.setMoonPhases(packet.totalMoonPhases);
    timeManager.setPaused(packet.timePaused);
}
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** Do not create an instance of UpdateTimeSettings on the client to send to the server. This packet is unidirectional (server-to-client), and the server will ignore it.
- **State Caching:** Do not retain a reference to a deserialized UpdateTimeSettings packet. Its data should be immediately copied to the relevant game state managers. Holding onto the packet object can lead to using stale data if a newer packet arrives.
- **Manual Serialization/Deserialization:** Do not call `serialize` or `deserialize` directly. These methods are designed to be invoked by the network protocol codecs as part of the Netty pipeline.

## Data Pipeline
The flow of this data is strictly from the server's game state to the client's game state.

> Flow:
> Server World State -> **UpdateTimeSettings** (Instance) -> Network Encoder -> TCP/IP -> Network Decoder -> **UpdateTimeSettings** (Instance) -> Packet Handler -> Client WorldTimeManager

