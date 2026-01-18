---
description: Architectural reference for TeleportToWorldMapPosition
---

# TeleportToWorldMapPosition

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class TeleportToWorldMapPosition implements Packet {
```

## Architecture & Concepts
The TeleportToWorldMapPosition class is a Data Transfer Object (DTO) that represents a specific, concrete network message within the Hytale protocol layer. As an implementation of the Packet interface, its primary role is to encapsulate the data required for a server to command a client to change the player's position on the world map.

This class is designed for extreme performance and low garbage collection overhead. It interacts directly with Netty's ByteBuf for serialization and deserialization, using little-endian byte order for its integer fields. The static constants, such as PACKET_ID and FIXED_BLOCK_SIZE, serve as critical metadata for the protocol's central packet dispatcher. This dispatcher uses the PACKET_ID to map an incoming byte stream to this specific class for decoding, a pattern known as a Packet Factory or Registry.

This object is not a service; it is inert data. It does not contain logic beyond what is necessary to convert itself to and from a byte stream.

### Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1. **Deserialization (Client-side):** The network layer's packet factory instantiates this object by calling the static `deserialize` method when it receives a byte stream with the corresponding PACKET_ID (245).
    2. **Serialization (Server-side):** Game logic on the server instantiates this object via its constructor when it needs to send a teleport command to a client.
- **Scope:** The object's lifetime is exceptionally short and bound to a single network event or game tick. It is created, processed, and then immediately becomes eligible for garbage collection. It should never be stored or reused.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no explicit cleanup methods. Holding long-term references to packet objects is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The class holds a simple, mutable state consisting of two public integer fields: x and y. It is a plain data structure with no caching, lazy initialization, or complex internal state.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created and used within the confines of a single thread, typically a Netty I/O worker thread or a main game loop thread. Concurrent modification of the x and y fields from multiple threads will lead to race conditions and unpredictable behavior.

**WARNING:** Do not share instances of this packet between threads. If data needs to be passed to another thread, either create a new copy or extract the primitive coordinate values.

## API Surface
The public contract is defined by the Packet interface and the static methods used for network I/O.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TeleportToWorldMapPosition(int, int) | constructor | O(1) | Creates a new packet with the specified coordinates. |
| getId() | int | O(1) | Returns the static network ID (245) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the x and y coordinates into the provided buffer. |
| deserialize(ByteBuf, int) | static TeleportToWorldMapPosition | O(1) | Reads 8 bytes from the buffer and constructs a new packet instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer is large enough to contain this packet. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload, which is 8 bytes. |

## Integration Patterns

### Standard Usage
This packet is typically handled by a registered packet processor or listener. The network layer decodes the stream, and the resulting object is passed to the relevant game system.

```java
// Example of a packet handler processing an incoming packet
void handleTeleport(TeleportToWorldMapPosition packet) {
    Player player = getLocalPlayer();
    WorldMap map = getGameUI().getWorldMap();

    // The packet's data is used to drive game logic
    map.setPlayerPosition(packet.x, packet.y);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to cache and re-use packet instances. They are cheap to create and should be treated as single-use, immutable-by-convention objects after creation.
- **Direct Instantiation on Client:** The client-side logic should never create this packet using `new`. It is a server-to-client command; clients only receive and deserialize it.
- **Ignoring Validation:** Bypassing the `validateStructure` check before calling `deserialize` on an untrusted buffer can lead to buffer overflow exceptions and potential server/client instability.

## Data Pipeline
The primary flow for this packet is from the server's game logic to the client's game logic, mediated by the network layer.

> **Flow (Server to Client):**
> Server Game Logic -> `new TeleportToWorldMapPosition(x, y)` -> Network Channel -> **serialize()** -> TCP/IP Stream -> Client Network Layer -> Packet Dispatcher (identifies ID 245) -> **deserialize()** -> Packet Handler -> Client Game Logic (Player Position Update)

