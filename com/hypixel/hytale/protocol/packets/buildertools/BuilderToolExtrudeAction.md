---
description: Architectural reference for BuilderToolExtrudeAction
---

# BuilderToolExtrudeAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class BuilderToolExtrudeAction implements Packet {
```

## Architecture & Concepts
The BuilderToolExtrudeAction class is a pure Data Transfer Object (DTO) that represents a specific, discrete player action within the game world. It is a fundamental component of the client-server communication protocol for in-game building mechanics.

This class does not contain any game logic. Its sole responsibility is to encapsulate the data required to describe an "extrude" operation—specifically, the target block's coordinates (x, y, z) and the direction of the extrusion (xNormal, yNormal, zNormal). It serves as a structured, serializable message that is sent from a game client to the server when a player uses the corresponding builder tool.

Architecturally, it is a leaf node in the protocol system. It is created, serialized, transmitted, deserialized, and consumed without holding long-term state or having complex dependencies. The static constants like PACKET_ID and FIXED_BLOCK_SIZE are critical metadata used by the higher-level network pipeline to identify, route, and parse the raw byte stream correctly.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Client-Side (Action Origination):** Instantiated by the client's input handling or game logic layer when the player performs an extrude action. The constructor is called with the world coordinates and normal vector of the targeted block face.
    2. **Server-Side (Action Reception):** Reconstructed by the network protocol decoder. The static `deserialize` method is invoked by a central packet dispatcher when an incoming byte buffer is identified with PACKET_ID 403.

- **Scope:** This object is **transient and extremely short-lived**. It exists only for the brief moment it takes to be serialized into a network buffer or, upon receipt, to be deserialized and passed to a handler for processing. It is not intended to be stored or referenced after the initial handling is complete.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by a packet handler on the receiving end. No systems retain a reference to it.

## Internal State & Concurrency
- **State:** The internal state is fully mutable, consisting of six public integer fields. While technically mutable, instances of this class should be treated as immutable after their initial creation or deserialization. The data represents a snapshot of a user action at a single point in time.

- **Thread Safety:** This class is **not thread-safe**. Its public, non-final fields are exposed to direct modification, and there are no internal synchronization mechanisms.

    **WARNING:** Access to an instance of BuilderToolExtrudeAction must be confined to a single thread, typically the network event loop thread (e.g., a Netty worker thread). Sharing this object across threads without explicit external locking will lead to race conditions and unpredictable behavior.

## API Surface
The public API is designed for serialization, deserialization, and data access, not for complex operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique network identifier for this packet type (403). |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into a Netty ByteBuf using little-endian byte order. |
| deserialize(ByteBuf, int) | BuilderToolExtrudeAction | O(1) | Static factory method. Decodes a new instance from a ByteBuf at a given offset. |
| computeSize() | int | O(1) | Returns the fixed size of the packet's payload in bytes (24). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Static utility to verify if a buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is almost never instantiated or managed directly by feature developers. It is processed by a dedicated packet handler on the server, which extracts the data to perform world modifications.

```java
// Example of a server-side packet handler
public class BuilderToolPacketHandler implements PacketHandler<BuilderToolExtrudeAction> {

    private final WorldService worldService;

    @Override
    public void handle(PlayerConnection connection, BuilderToolExtrudeAction packet) {
        // The packet object is provided by the network layer
        // Extract data and pass to a dedicated game system
        WorldCoordinate target = new WorldCoordinate(packet.x, packet.y, packet.z);
        Vector3i normal = new Vector3i(packet.xNormal, packet.yNormal, packet.zNormal);

        // Delegate the actual game logic to a world modification service
        worldService.performExtrude(connection.getPlayer(), target, normal);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify the fields of a packet after it has been received. It represents a past event and should be treated as immutable.
- **Object Caching/Re-use:** Do not cache or reuse packet instances. They are lightweight and should be created for each distinct action to avoid state corruption.
- **Manual Serialization:** Avoid calling `serialize` or `deserialize` outside of the core network protocol handlers. The network engine is responsible for the entire serialization pipeline.

## Data Pipeline
The flow of this data object is linear and unidirectional from the client to the server.

> Flow:
> Client Input System -> **new BuilderToolExtrudeAction()** -> Network Encoder (`serialize`) -> TCP/IP Stack -> Server Network Decoder (`deserialize`) -> Packet Handler -> World Simulation Service

