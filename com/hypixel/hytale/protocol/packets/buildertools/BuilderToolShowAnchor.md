---
description: Architectural reference for BuilderToolShowAnchor
---

# BuilderToolShowAnchor

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolShowAnchor implements Packet {
```

## Architecture & Concepts
The BuilderToolShowAnchor is a simple, fixed-size Data Transfer Object (DTO) that operates within the Hytale network protocol layer. Its sole purpose is to communicate a specific 3D world coordinate from a server to a client. This packet is part of the "Builder Tools" feature set, used to command the client to render a visual indicator, or "anchor," at the specified location.

As a concrete implementation of the Packet interface, it is designed for high-performance serialization and deserialization. The class structure is intentionally flat, containing only primitive integer fields for coordinates. The presence of static constants like PACKET_ID, FIXED_BLOCK_SIZE, and IS_COMPRESSED indicates its role as a well-defined message in a low-level, performance-critical protocol. It does not contain any game logic; it is purely a data container.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** Instantiated directly via its constructor, `new BuilderToolShowAnchor(x, y, z)`, when game logic dictates that a builder tool anchor should be displayed on a client's screen.
    - **Receiving Peer (Client):** Instantiated by the network protocol layer via the static `deserialize` factory method. This occurs when an incoming network buffer is decoded and the packet ID 415 is identified. Application code should never call `deserialize` directly.

- **Scope:** Extremely short-lived and transient. An instance of this packet exists only for the brief moment it is being serialized for network transmission or after it has been deserialized and is being processed by a packet handler.

- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by a packet handler. There are no internal mechanisms for cleanup, as it holds no managed resources.

## Internal State & Concurrency
- **State:** The state is fully mutable and consists of three public integer fields: x, y, and z. The object's state is established at creation and is not expected to change during its lifecycle. It performs no caching and has no internal logic beyond data representation.

- **Thread Safety:** This class is **not thread-safe**. It is a simple POJO with no synchronization mechanisms. This is by design. Packet objects are intended to be processed sequentially by a single network thread (e.g., a Netty event loop) or handed off to a main game thread in a thread-safe manner. Concurrent access from multiple threads will lead to race conditions and is an unsupported usage pattern.

## API Surface
The public API is focused entirely on protocol-level operations for serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (415). |
| serialize(ByteBuf) | void | O(1) | Writes the x, y, and z fields into the provided buffer using little-endian byte order. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes (12). |
| deserialize(ByteBuf, int) | BuilderToolShowAnchor | O(1) | **Static Factory.** Reads 12 bytes from the buffer and constructs a new packet instance. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is handled by a client-side packet processor. The handler extracts the coordinate data and forwards it to the appropriate rendering or game system to manage the visual representation of the builder tool anchor.

```java
// Example of a client-side packet handler
public void handle(BuilderToolShowAnchor packet) {
    // Extract data from the transient packet object
    int worldX = packet.x;
    int worldY = packet.y;
    int worldZ = packet.z;

    // Pass primitive data to a long-lived service
    // The 'packet' object is now out of scope and can be garbage collected
    this.renderingSystem.displayBuilderAnchorAt(worldX, worldY, worldZ);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of BuilderToolShowAnchor in collections or as member variables in long-lived services. This is a memory leak risk. Extract the primitive data you need and discard the packet object.
- **State Modification:** Do not modify the public fields of a packet after it has been received. While technically possible, it violates the contract of it being a snapshot of data at a point in time and can lead to unpredictable behavior.
- **Manual Deserialization:** Application code should never call `deserialize` directly. This method is strictly for use by the low-level protocol decoder. Interacting with network buffers at the application level is a critical error.

## Data Pipeline
The flow of this data object is unidirectional from the server's game state to the client's renderer.

> **Server Flow:**
> Game Logic Event -> `new BuilderToolShowAnchor(x, y, z)` -> Protocol Encoder (`serialize`) -> TCP/IP Stack

> **Client Flow:**
> TCP/IP Stack -> Netty ByteBuf -> Protocol Decoder (`deserialize`) -> **BuilderToolShowAnchor Instance** -> Packet Handler -> Rendering Engine

