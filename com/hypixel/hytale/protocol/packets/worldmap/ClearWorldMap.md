---
description: Architectural reference for ClearWorldMap
---

# ClearWorldMap

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class ClearWorldMap implements Packet {
```

## Architecture & Concepts
The ClearWorldMap packet is a stateless command object within the Hytale network protocol. It serves a single, critical function: to instruct a client to completely purge its local cache and in-memory representation of the world map.

Unlike data-carrying packets, ClearWorldMap is a *signal*. Its transmission and reception are the entire message; it contains no payload or variable fields. This design makes it an extremely lightweight and efficient mechanism for forcing a state reset on the client. It is typically dispatched by the server during significant world state transitions, such as inter-dimensional travel, teleportation to a completely new region, or to recover from a client-side desynchronization event. It acts as a control plane instruction, managing the lifecycle of client-side map data rather than transferring the data itself.

## Lifecycle & Ownership
- **Creation:** On the server, an instance is created by game logic controllers when a map reset is required for a specific client or group of clients. On the client, an instance is created exclusively by the protocol's deserialization layer when an incoming network buffer is identified with Packet ID 242.
- **Scope:** This object is ephemeral and has a very short lifespan. It exists only for the duration of a single transaction: serialization and transmission on the server, or deserialization and processing on the client. It is not designed to be stored or referenced after its initial handling.
- **Destruction:** The object is eligible for garbage collection immediately after it has been processed by a packet handler on the client, or after it has been serialized into a ByteBuf on the server. There is no persistent ownership.

## Internal State & Concurrency
- **State:** ClearWorldMap is a stateless and immutable object. It contains no instance fields and its identity is defined entirely by its class type and packet ID. All instances are functionally identical.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely passed between threads without synchronization. In a typical Netty-based architecture, it is deserialized on an I/O worker thread and subsequently passed to a main game logic thread for processing.

## API Surface
The public contract is minimal, primarily fulfilling the requirements of the Packet interface and providing static methods for the protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet, 242. |
| serialize(ByteBuf) | void | O(1) | Writes nothing to the buffer. This packet has a zero-byte payload. |
| deserialize(ByteBuf, int) | ClearWorldMap | O(1) | Creates a new instance. Consumes zero bytes from the buffer. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a basic bounds check. Always returns OK if the buffer is valid. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct instantiation or use by general application code. Its lifecycle is managed entirely by the network protocol stack. Client-side systems subscribe to this packet type via a central dispatcher.

```java
// Example of a client-side packet handler
public class WorldMapPacketHandler {

    private final WorldMapManager mapManager;

    public void handle(ClearWorldMap packet) {
        // The receipt of this packet triggers the system action.
        // No data from the packet itself is needed.
        this.mapManager.purgeAllMapData();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** Do not create instances of ClearWorldMap on the client for any reason. The client's role is to receive and handle this packet, not to send it. Sending this packet to the server would be a protocol violation.
- **State Assumption:** Do not attempt to read data from this packet. It is a pure command and carries no state. Any logic that depends on a payload will fail.
- **Ignoring the Command:** Failure to handle this packet will lead to severe client-side state corruption, where the client may render stale or incorrect map data until a full client restart.

## Data Pipeline
The flow for this packet is unidirectional from server to client. It acts as a control signal that flushes a client-side data cache.

> Flow:
> Server Game Event -> **new ClearWorldMap()** -> Packet Serializer -> TCP/IP Stack -> Client Network Receiver -> Packet Deserializer -> Packet Dispatcher -> WorldMapManager.purgeAllMapData()

