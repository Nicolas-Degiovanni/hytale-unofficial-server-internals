---
description: Architectural reference for AssetEditorAssetUpdated
---

# AssetEditorAssetUpdated

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorAssetUpdated implements Packet {
```

## Architecture & Concepts
The AssetEditorAssetUpdated class is a message contract within the Hytale network protocol, designed for high-performance, real-time synchronization of game assets. It is not a service or manager, but rather a transient Data Transfer Object (DTO) that represents the state of a single asset being updated by the in-game asset editor.

This packet is a critical component of the live-editing development workflow. When a developer modifies a texture, model, or configuration file, the server-side editor constructs and serializes an AssetEditorAssetUpdated packet. This packet is then broadcast to connected clients, which deserialize it and hot-reload the corresponding asset in memory without requiring a client restart.

The binary layout is heavily optimized for speed and minimal overhead. It employs a direct memory layout strategy consisting of a fixed-size header and a variable-size data block. The header contains a bitfield for nullable members and integer offsets pointing to the location of variable-length fields within the packet's payload. This design allows for extremely fast validation and deserialization, as the decoder can seek directly to the required data without parsing the entire stream.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Sending):** Instantiated directly by server-side application logic, typically within an Asset Editor service, when an asset file is saved. The constructor `new AssetEditorAssetUpdated(path, data)` is used to populate it with the asset's identifier and new binary content.
    - **Inbound (Receiving):** Instantiated by a low-level protocol decoder, such as a Netty ChannelInboundHandler, upon receiving a network buffer with the corresponding packet ID (326). The static factory method `deserialize` is invoked to construct the object from the raw byte buffer.

- **Scope:** The lifecycle of an AssetEditorAssetUpdated instance is exceptionally brief. It is designed to be a stateless message, existing only for the duration of its travel through the network processing pipeline.

- **Destruction:** The object becomes eligible for garbage collection immediately after being processed by its designated handler (e.g., an asset management service) or after its contents have been fully written to an outbound network buffer. It holds no persistent state and is not managed by any container.

## Internal State & Concurrency
- **State:** The class is a mutable data container. Its public fields, AssetPath and data, can be modified after instantiation. This design prioritizes performance and ease of use within the protocol layer over strict immutability.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads without explicit, external synchronization.** It is designed to be created, populated, and processed within the confines of a single thread, such as a Netty event loop. Concurrent modification of the path or data fields will lead to race conditions and unpredictable behavior.

## API Surface
The public contract is focused on serialization, deserialization, and validation within the protocol framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorAssetUpdated | O(N) | **Static Factory.** Constructs an object by reading from a Netty ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into a Netty ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | ValidationResult | O(1) | **Static.** Performs a low-cost check on a buffer to verify that it contains a structurally valid packet without full deserialization. |
| computeSize() | int | O(1) | Calculates the final size in bytes of the serialized packet payload. Essential for pre-allocating buffers. |

*N = size of the asset data byte array.*

## Integration Patterns

### Standard Usage
The primary interaction pattern involves either creating a packet to be sent or receiving and processing a deserialized packet from the network layer.

**Sending an Asset Update (Server-Side Logic)**
```java
// Example: A service prepares to notify clients of a texture update.
AssetPath texturePath = new AssetPath("hytale", "textures/blocks/stone.png");
byte[] newTextureData = Files.readAllBytes(Paths.get("..."));

// 1. Instantiate the packet with the asset data.
AssetEditorAssetUpdated packet = new AssetEditorAssetUpdated(texturePath, newTextureData);

// 2. Pass the packet to the network channel for serialization and transmission.
playerConnection.sendPacket(packet);
```

**Receiving an Asset Update (Client-Side Handler)**
```java
// Example: A handler receives the deserialized packet from the protocol decoder.
public void handleAssetUpdate(AssetEditorAssetUpdated packet) {
    // 1. Extract the data.
    AssetPath path = packet.path;
    byte[] data = packet.data;

    // 2. Dispatch to the appropriate system for hot-reloading.
    AssetManager assetManager = ServiceRegistry.get(AssetManager.class);
    assetManager.hotReloadAsset(path, data);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not hold references to and re-use packet instances for multiple transmissions. They should be treated as single-use messages. Create a new instance for each distinct update.
- **Manual Serialization:** Avoid calling `serialize` or `deserialize` outside of the core network protocol handlers. The network layer is responsible for managing the byte buffers and the serialization lifecycle.
- **Concurrent Access:** Never modify a packet instance from one thread while another thread is reading it or serializing it to a network buffer.

## Data Pipeline
The AssetEditorAssetUpdated packet is a data-flow component. Its journey through the system is linear and unidirectional.

**Outbound Flow (Server to Client):**
> Asset Editor Service -> **AssetEditorAssetUpdated (Instance)** -> Protocol Encoder -> Netty Channel -> Raw Bytes on Wire

**Inbound Flow (Client from Server):**
> Raw Bytes on Wire -> Netty Channel -> Protocol Decoder -> **AssetEditorAssetUpdated (Instance)** -> Client-Side Event Bus -> AssetManager -> Asset Hot-Reload

