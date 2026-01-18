---
description: Architectural reference for BuilderToolLaserPointer
---

# BuilderToolLaserPointer

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolLaserPointer implements Packet {
```

## Architecture & Concepts
The BuilderToolLaserPointer class is a Data Transfer Object (DTO) that represents a network packet. Its sole purpose is to transmit the state of a temporary, client-side visual effect from the server to the client. This packet is a fundamental component of the "Builder Tools" feature set, allowing players with appropriate permissions to see a visual representation of where another builder is aiming in the world.

Architecturally, this class acts as a contract between the server's game logic and the client's rendering engine. By encapsulating the laser's properties—start point, end point, color, and duration—it decouples the server from any rendering-specific implementation details. The server simply broadcasts this state change, and it is the client's responsibility to interpret this data and render the corresponding visual effect.

This packet is defined by a fixed-size data structure, as indicated by the constant FIXED_BLOCK_SIZE. This design choice prioritizes performance and predictability in network processing, as the size of the packet is known ahead of time, eliminating the need for variable-length decoding.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated directly via its constructor (`new BuilderToolLaserPointer(...)`) by the server's game logic when a player action triggers the laser pointer effect.
    - **Client-Side:** Instantiated by the network protocol layer, specifically through the static `deserialize` method, when an incoming network buffer with PACKET_ID 419 is processed.

- **Scope:** The object's lifetime is exceptionally short and scoped to a single network transaction. Once serialized and sent by the server, the original object is eligible for garbage collection. On the client, once the object is deserialized and its data is consumed by the appropriate handler (e.g., the rendering system), it is no longer referenced and becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The class is a mutable container for primitive data types. All of its fields are public, allowing for direct modification after instantiation. However, in practice, it should be treated as an immutable object after its initial population on the server or deserialization on the client.

- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it inherently unsafe for concurrent access without external synchronization. It is designed to be created, processed, and discarded within the confines of a single thread, such as a Netty network thread or the main game loop.

    **WARNING:** Modifying a BuilderToolLaserPointer instance from one thread while another thread is reading from or serializing it will lead to race conditions and unpredictable application behavior.

## API Surface
The primary contract is defined by its static deserialization methods and its instance-level serialization method, which are used by the core protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolLaserPointer | O(1) | **[Client]** Constructs a new packet by reading a fixed block of 36 bytes from the provided ByteBuf. |
| serialize(buf) | void | O(1) | **[Server]** Writes the packet's 36 bytes of state into the provided ByteBuf. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid packet. |
| computeSize() | int | O(1) | Returns the constant size of the packet, 36 bytes. |

## Integration Patterns

### Standard Usage
This packet is almost never interacted with directly by feature developers. The protocol engine handles its lifecycle. The following demonstrates how the client-side network layer would process the raw data.

```java
// Client-side network handler context
ByteBuf incomingBuffer = ...; // Buffer received from network
int packetOffset = ...; // Offset within the buffer

// 1. Validate that a full packet can be read
ValidationResult result = BuilderToolLaserPointer.validateStructure(incomingBuffer, packetOffset);
if (result.isOk()) {
    // 2. Deserialize the data into a structured object
    BuilderToolLaserPointer packet = BuilderToolLaserPointer.deserialize(incomingBuffer, packetOffset);

    // 3. Dispatch to the game engine for processing
    gameClient.getPacketHandler().handle(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to cache and re-use BuilderToolLaserPointer instances. They are cheap to create and are not designed for pooling. Modifying and re-sending a previously used packet can lead to data corruption.
- **Client-Side Instantiation:** Do not use `new BuilderToolLaserPointer()` on the client. Client-side packets must only be created via the `deserialize` method to ensure they reflect the true state sent by the server.
- **Cross-Thread Modification:** Do not deserialize a packet on a network thread and pass the reference to a game thread without ensuring all mutations are complete. The object should be treated as read-only once it leaves the network layer.

## Data Pipeline
The flow of this data object is unidirectional from server to client.

> **Outbound (Server) Flow:**
> Player Input -> Server Game Logic -> `new BuilderToolLaserPointer(...)` -> **Packet::serialize** -> Network Encoder -> TCP/UDP Socket

> **Inbound (Client) Flow:**
> TCP/UDP Socket -> Network Decoder -> **BuilderToolLaserPointer::deserialize** -> Packet Handler -> Rendering System -> Visual Effect Displayed

