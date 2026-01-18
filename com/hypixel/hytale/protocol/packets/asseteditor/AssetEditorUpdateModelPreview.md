---
description: Architectural reference for AssetEditorUpdateModelPreview
---

# AssetEditorUpdateModelPreview

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class AssetEditorUpdateModelPreview implements Packet {
```

## Architecture & Concepts

The AssetEditorUpdateModelPreview class is a **Packet**, a specialized Data Transfer Object (DTO) designed for network communication. It does not contain business logic; its sole purpose is to serve as a structured container for data sent between the Hytale client's Asset Editor and a rendering or preview service.

This packet encapsulates all the necessary information to refresh the live preview of a model, including the model data itself, an associated block type, the asset's canonical path, and camera settings.

Its architecture is defined by a highly optimized, custom binary serialization protocol. The protocol is designed for performance and low memory overhead, avoiding reflection-based or general-purpose serialization libraries. Key features of this binary format include:

*   **Fixed-Size Header:** A predictable block at the start of the packet data (`FIXED_BLOCK_SIZE`) contains essential metadata.
*   **Nullable Bit Field:** The very first byte of the packet is a bitmask indicating which of the nullable fields are present in the payload. This avoids wasting space for null data.
*   **Variable Data Offsets:** The fixed-size header contains integer offsets pointing to the location of variable-sized data (like Model or AssetPath) within the packet's data block. This allows for efficient, direct-memory access during deserialization without parsing the entire payload sequentially.

This design is critical for the interactive nature of the asset editor, where frequent, low-latency updates are required as a user manipulates a model.

## Lifecycle & Ownership

-   **Creation:** An instance is created on the sending side (the client) whenever a user's action in the Asset Editor necessitates a preview update. On the receiving side, a new instance is created by the network layer's packet factory, specifically via the static `deserialize` method, which reads from a raw network buffer.
-   **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of a single network operation: creation, serialization, network transit, deserialization, and consumption by a handler.
-   **Destruction:** As a simple Java object with no native resources, it is managed entirely by the Garbage Collector. Once the receiving system has processed the data, the packet instance is dereferenced and becomes eligible for collection.

## Internal State & Concurrency

-   **State:** The class is a mutable data container. All of its fields are public and can be modified directly after instantiation. This design choice prioritizes performance by allowing the serialization and deserialization logic to manipulate the object's state with minimal overhead. It holds no logic, only data.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed by a single thread, typically a network I/O thread (like a Netty event loop) or a main game thread. Sharing instances across threads without explicit, external synchronization is an anti-pattern and will result in data corruption and unpredictable behavior.

## API Surface

The public contract is dominated by static methods for serialization and validation, reflecting its role as a data structure rather than a service.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorUpdateModelPreview | O(N) | **Primary Constructor.** Rehydrates a packet object from a raw ByteBuf. N is the size of the variable-length data. |
| serialize(buf) | void | O(N) | **Primary Producer.** Writes the object's state into a ByteBuf according to the custom binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a low-cost integrity check on a buffer before attempting full deserialization. Critical for protocol stability and security. |
| computeSize() | int | O(1) | Calculates the final serialized size of the packet based on its current state. Useful for pre-allocating buffers. |
| getId() | int | O(1) | Returns the static packet identifier (355), used by the protocol dispatcher to route the packet to the correct handler. |

## Integration Patterns

### Standard Usage

This packet is intended to be handled by the network protocol layer. A developer would typically interact with it within a packet handler after it has been fully deserialized from the network channel.

```java
// Example of a hypothetical packet handler
public class AssetEditorPacketHandler {
    private final PreviewRenderer previewRenderer;

    public void handlePacket(AssetEditorUpdateModelPreview packet) {
        // The packet is fully formed and ready for consumption.
        // Do not modify the packet here; treat it as immutable.
        if (packet.model != null) {
            previewRenderer.updateModel(packet.model);
        }
        if (packet.camera != null) {
            previewRenderer.updateCamera(packet.camera);
        }
        // ... and so on for other fields.
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Reuse:** Do not modify and re-send the same packet instance. The serialization logic is not designed for this, and it can lead to subtle bugs. Always create a new packet for each distinct message.
-   **Manual Buffer Manipulation:** Never attempt to read or write the binary format manually. The layout is complex, involving bitmasks and relative offsets. Always use the provided `serialize` and `deserialize` methods.
-   **Cross-Thread Sharing:** Do not pass a packet instance to another thread for processing without creating a defensive copy via the `clone` method. Its mutable nature makes it inherently unsafe for concurrent access.

## Data Pipeline

The AssetEditorUpdateModelPreview packet is a key component in the real-time asset editing data flow. Its journey is linear and transactional.

> Flow:
> User Action in Asset Editor UI -> Event Handler creates **AssetEditorUpdateModelPreview** -> `serialize()` -> Network Channel (e.g., Netty) -> Raw TCP/UDP Data -> Receiving Network Channel -> Packet Dispatcher uses `PACKET_ID` -> `deserialize()` -> **AssetEditorUpdateModelPreview** is passed to Handler -> Preview Rendering System consumes data -> Rendered Frame is updated

