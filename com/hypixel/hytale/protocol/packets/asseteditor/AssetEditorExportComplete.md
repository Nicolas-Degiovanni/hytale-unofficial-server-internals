---
description: Architectural reference for AssetEditorExportComplete
---

# AssetEditorExportComplete

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorExportComplete implements Packet {
```

## Architecture & Concepts
The AssetEditorExportComplete packet is a Data Transfer Object (DTO) operating within the Hytale network protocol layer. Its primary function is to signal the successful completion of an asset export operation initiated within the game's asset editor.

This packet acts as the definitive confirmation message, sent from the system performing the export (e.g., a dedicated asset editor service or the game client itself) to a receiving system (e.g., the game server or another client). It contains a list of references to the newly created or modified assets, allowing the recipient to acknowledge the changes and potentially trigger subsequent actions, such as reloading assets or updating internal registries.

It is a fundamental component of the live-editing and content creation pipeline, ensuring that asset changes are communicated reliably across the distributed game environment.

## Lifecycle & Ownership
- **Creation:** An instance of AssetEditorExportComplete is created by the asset processing service immediately after a batch of assets has been successfully written to storage. The service populates the *assets* field with an array of TimestampedAssetReference objects that point to the exported files.
- **Scope:** The lifecycle of this object is exceptionally short and bound to a single network transaction. It exists only for the duration of serialization, network transit, and deserialization on the receiving end.
- **Destruction:** Once the packet has been deserialized and its data consumed by a packet handler, it becomes eligible for garbage collection. There is no manual resource management associated with this object.

## Internal State & Concurrency
- **State:** The internal state consists of a single, nullable array of TimestampedAssetReference objects. The state is mutable by design, intended to be populated once upon creation before being passed to the network layer. After this point, it should be treated as immutable.
- **Thread Safety:** This class is **not thread-safe**. As a simple DTO, it contains no internal synchronization mechanisms. It is designed to be created, populated, and serialized within a single-threaded context, such as a network I/O thread or the main game loop. Concurrent access to the *assets* field will lead to unpredictable behavior and data corruption.

## API Surface
The public contract is defined by the Packet interface and the serialization logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorExportComplete(assets) | constructor | O(1) | Constructs a new packet with the specified asset references. |
| serialize(ByteBuf) | void | O(N) | Encodes the packet's state into the provided Netty ByteBuf. Throws ProtocolException on validation failure. |
| deserialize(ByteBuf, int) | AssetEditorExportComplete | O(N) | Decodes a packet from the ByteBuf. Throws ProtocolException if the data is malformed or violates constraints. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the packet. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a valid packet structure without full deserialization. |

*N = number of elements in the assets array.*

## Integration Patterns

### Standard Usage
This packet is exclusively handled by the network protocol engine. A packet handler on the receiving end will listen for this packet type, deserialize it, and forward the list of asset references to the appropriate system for processing.

```java
// Example of a packet handler consuming the packet
public void handle(AssetEditorExportComplete packet) {
    if (packet.assets != null) {
        AssetUpdateService service = context.getService(AssetUpdateService.class);
        service.processNewAssetReferences(packet.assets);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification After Queuing:** Never modify the *assets* array after the packet instance has been passed to the network manager for serialization. The serialization may occur on a different thread at a later time, leading to a race condition.
- **Object Re-use:** Do not attempt to cache and re-use AssetEditorExportComplete instances. They are lightweight objects that should be created for each new export event.
- **Manual Serialization:** Avoid calling **serialize** or **deserialize** directly. These methods are low-level and intended for use only by the core protocol codec.

## Data Pipeline
The packet's data flows from the asset editing environment, across the network, and into the game state of a remote peer.

> Flow:
> Asset Editor Service -> **AssetEditorExportComplete** (Instantiation) -> Protocol Codec (Serialization) -> Network Layer (Transmission) -> Protocol Codec (Deserialization) -> Packet Handler -> Asset Manager (State Update)

