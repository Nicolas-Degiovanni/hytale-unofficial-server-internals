---
description: Architectural reference for AssetEditorExportAssetPart
---

# AssetEditorExportAssetPart

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorExportAssetPart implements Packet {
```

## Architecture & Concepts
The AssetEditorExportAssetPart class is a low-level Data Transfer Object (DTO) within the Hytale network protocol layer. It does not contain any business logic; its sole purpose is to represent a single, serialized chunk of a larger asset being transferred from a client-side asset editor to a server or peer.

This packet is a fundamental building block for the real-time asset editing and synchronization feature. The system is designed to handle potentially large assets (models, textures, configurations) by breaking them into manageable byte array segments. This class encapsulates one such segment.

Its design prioritizes wire-efficiency and deserialization performance. It employs a custom serialization format that uses a leading bitfield to denote nullable fields and variable-length integers (VarInt) to encode the payload size, minimizing network overhead. The class is effectively a data schema definition for packet ID 344.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer:** An instance is created directly using its constructor, `new AssetEditorExportAssetPart(byteArray)`, by a higher-level asset streaming service that is responsible for chunking the source asset file.
    - **Receiving Peer:** An instance is never created with `new`. It is exclusively instantiated by the network protocol decoder which invokes the static `deserialize` factory method on an incoming Netty ByteBuf.

- **Scope:** Extremely short-lived and transient. An instance exists only within the scope of a single network read or write operation. On the receiving end, it lives only long enough for a packet handler to extract its byte array payload and pass it to an asset reassembly buffer.

- **Destruction:** The object is immediately eligible for garbage collection after its data has been processed by a packet handler. No long-term references should ever be held to instances of this class.

## Internal State & Concurrency
- **State:** The class holds a single mutable state field: the `part` byte array. While technically mutable, instances should be treated as immutable value objects after their initial creation or deserialization. Modifying the state of a packet after it has been processed is an anti-pattern.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be confined to a single thread, typically a Netty event loop thread. All serialization, deserialization, and handling must occur on the same thread to prevent race conditions and memory visibility issues. Do not share instances of this packet across threads.

## API Surface
The primary contract is for serialization and deserialization, orchestrated by the network layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | AssetEditorExportAssetPart | O(N) | **[Static Factory]** Reads from a buffer and constructs a new packet instance. N is the size of the asset part. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the internal state of the packet into the provided buffer according to the protocol specification. N is the size of the asset part. |
| computeSize() | int | O(1) | Calculates the final serialized size of the packet in bytes without performing the actual serialization. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **[Static]** Performs a lightweight check on a buffer to validate if it contains a structurally correct packet without full deserialization. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game logic developers. It is processed by the network layer and its payload is passed to specialized asset management systems. A hypothetical packet handler would extract the data for reassembly.

```java
// Executed on a network thread by a packet dispatcher
public class AssetEditorPacketHandler {

    private final AssetReassemblyService reassemblyService;

    public void handleAssetPart(AssetEditorExportAssetPart packet) {
        if (packet.part == null) {
            // Handle potential empty part, though unlikely
            return;
        }
        
        // The handler's only job is to extract the payload and
        // delegate it to the system responsible for re-combining the asset.
        this.reassemblyService.appendChunk(packet.part);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of this packet in collections or as member variables. This constitutes a memory leak, as they hold onto potentially large byte arrays. Extract the data and discard the packet object.
- **State Mutation:** Do not modify the `part` field after the packet has been deserialized. Treat it as a read-only data record.
- **Cross-Thread Access:** Never pass this object from the network thread to another thread. Instead, perform a defensive copy of the `part` byte array and pass the copy.

## Data Pipeline
The AssetEditorExportAssetPart serves as a transient data container in the asset streaming pipeline.

> **Sending Peer Flow:**
> Full Asset Data -> Asset Chunking Service -> **new AssetEditorExportAssetPart(chunk)** -> `serialize()` -> Network ByteBuf -> TCP Stream

> **Receiving Peer Flow:**
> TCP Stream -> Network ByteBuf -> Protocol Decoder -> `deserialize()` -> **AssetEditorExportAssetPart** -> Packet Handler -> Asset Reassembly Service

