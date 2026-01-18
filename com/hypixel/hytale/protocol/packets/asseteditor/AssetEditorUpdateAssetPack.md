---
description: Architectural reference for AssetEditorUpdateAssetPack
---

# AssetEditorUpdateAssetPack

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorUpdateAssetPack implements Packet {
```

## Architecture & Concepts
The AssetEditorUpdateAssetPack class is a Data Transfer Object (DTO) that represents a single, specific message within the Hytale network protocol. It is not a service or manager; it is a pure data container with no inherent logic beyond its own serialization and deserialization.

Its primary role is to encapsulate the data required to update an asset pack from a client to a server, typically originating from the in-game asset editing tools. It acts as a strict data contract between the client and server, defining the precise binary structure for transmitting an asset pack's unique identifier and its associated manifest.

The class employs a highly optimized, low-level binary format for network transport. Key architectural features include:
*   **ID-Based Dispatch:** The static PACKET_ID field (315) allows the protocol's multiplexer to quickly identify the byte stream and delegate parsing to this specific class.
*   **Offset-Based Layout:** The binary layout uses a fixed-size header containing a null-bit field and offsets to a variable-data section. This allows for efficient handling of optional fields (id, manifest) and variable-length data without wasting bandwidth, a common pattern in high-performance game networking.
*   **Direct Buffer Manipulation:** All serialization and deserialization logic operates directly on Netty ByteBuf instances. This avoids intermediate object allocations and minimizes GC pressure, which is critical for a real-time network layer.

This packet is a fundamental component of the live asset editing pipeline, enabling developers and creators to push changes to a running server without a full restart.

## Lifecycle & Ownership
-   **Creation:**
    -   **Outbound (Sending):** An instance is created using its constructor, for example, `new AssetEditorUpdateAssetPack(id, manifest)`, by a higher-level system that is responding to a user action in the asset editor. This object is then passed to the network channel for transmission.
    -   **Inbound (Receiving):** An instance is created exclusively by the network protocol layer. The static `deserialize` factory method is invoked by a protocol decoder when an incoming byte stream is identified as packet 315. Application code should never call `deserialize` directly.
-   **Scope:** The object's lifetime is exceptionally brief and transient. It exists only for the time it takes to be processed by the network pipeline. On the sending side, it is serialized and can be immediately garbage collected. On the receiving side, it is deserialized, passed to a single handler, and then becomes eligible for garbage collection once the handler completes its work.
-   **Destruction:** Managed entirely by the Java Garbage Collector. The class holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is a simple, mutable data structure. Its state consists of the nullable `id` and `manifest` fields. It does not cache any data or maintain any state beyond the values provided at construction.
-   **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. It is intended to be created, populated, and processed within a single thread, typically a Netty I/O worker thread. Any attempt to modify an instance from multiple threads will result in race conditions and undefined behavior. External synchronization must be provided if multi-threaded access is unavoidable.

## API Surface
The primary contract is for the network protocol itself, not for general application use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Serializes the object's state into the target ByteBuf according to the defined binary protocol. Throws ProtocolException on data validation failures. |
| deserialize(ByteBuf, int) | static AssetEditorUpdateAssetPack | O(N) | Deserializes bytes from the source ByteBuf into a new AssetEditorUpdateAssetPack instance. Throws ProtocolException if the data is malformed. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only structural validation of the data in a buffer. This is a crucial security and stability method used to reject malformed packets early, before attempting a full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will occupy on the wire when serialized. Used by the protocol layer for buffer pre-allocation. |

## Integration Patterns

### Standard Usage
A developer will typically interact with this packet within a network event handler. The network layer is responsible for decoding the byte stream into the object. The developer's code consumes the object.

**WARNING:** The following example is conceptual. You do not call deserialize directly; it is done by the network pipeline before your handler is invoked.

```java
// In a network message handler (e.g., a Netty ChannelInboundHandler)
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    if (msg instanceof AssetEditorUpdateAssetPack) {
        AssetEditorUpdateAssetPack packet = (AssetEditorUpdateAssetPack) msg;

        // Safely access the packet data
        String assetPackId = packet.id;
        AssetPackManifest newManifest = packet.manifest;

        // Pass the data to the appropriate game system
        AssetService service = context.getService(AssetService.class);
        service.updateAssetPack(assetPackId, newManifest);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Serialization:** Do not call `serialize` or `deserialize` in application-level code. These methods are tightly coupled to the protocol's framing and encoding pipeline. Bypassing the pipeline will result in corrupted network data.
-   **Instance Reuse:** Do not modify a packet instance after passing it to the network channel for sending. The network layer may perform serialization on a different thread, leading to a race condition. Always create a new packet instance for each distinct message.
-   **Ignoring Validation:** On a publicly exposed server, the network layer must use `validateStructure` before attempting to `deserialize`. Bypassing this check can expose the server to denial-of-service attacks by sending packets with invalid lengths or offsets, causing exceptions or excessive memory allocation.

## Data Pipeline
The AssetEditorUpdateAssetPack serves as a data payload that flows through the network stack. It is instantiated on one end of the connection and reconstituted on the other.

> Flow:
> User Action in Asset Editor -> **AssetEditorUpdateAssetPack (Instance created)** -> Network Channel -> Protocol Encoder (calls `serialize`) -> Raw Bytes on Wire -> Network Transport -> Protocol Decoder (calls `deserialize`) -> **AssetEditorUpdateAssetPack (Instance reconstituted)** -> Network Event Handler -> Asset Management System

