---
description: Architectural reference for AssetEditorExportAssets
---

# AssetEditorExportAssets

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorExportAssets implements Packet {
```

## Architecture & Concepts
AssetEditorExportAssets is a network **Packet** that functions as a Data Transfer Object. Its sole purpose is to encapsulate a request to export a collection of assets from the server-side asset editor. It represents a specific, well-defined message within the Hytale network protocol, identified by the static packet ID 342.

This class is a fundamental component of the protocol layer, acting as the structured data contract between the client and the server for asset management operations. It does not contain any business logic; its responsibilities are strictly limited to serialization, deserialization, and data validation according to the protocol's binary format. The design emphasizes efficiency and strict validation, with hard-coded limits on array sizes to prevent denial-of-service attacks via oversized packets.

The binary layout is optimized for size, using a single byte as a bitfield to denote the presence of its nullable fields, followed by a VarInt for the array length. This ensures that null payloads consume minimal network bandwidth.

## Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1.  **Ingress:** The network protocol decoder instantiates the object by calling the static **deserialize** method when an incoming network buffer with packet ID 342 is processed.
    2.  **Egress:** The application logic (e.g., an asset management service preparing a request) creates an instance using its constructor, populates the **paths** array, and passes it to the network encoder for serialization.
- **Scope:** Transient and short-lived. An instance exists only for the duration of a single network event processing cycle. It is created, processed by a handler, and then becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required. Ownership is never transferred long-term; it is passed by reference to a handler and immediately discarded.

## Internal State & Concurrency
- **State:** Mutable. The primary state is the **paths** field, which is a nullable array of AssetPath objects. The object's state is fully defined by its constructor arguments or the data deserialized from a network buffer. It performs no caching and has no internal state beyond its public fields.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread. Concurrent modification of the **paths** array from multiple threads will result in unpredictable behavior and data corruption. All synchronization must be handled externally by the calling context.

## API Surface
The public API is exclusively for protocol-level operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorExportAssets | O(N) | **Static Factory.** Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the protocol specification. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the serialized object will occupy. Critical for buffer pre-allocation. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Performs a read-only check of a buffer to ensure it contains a valid representation of this packet. |
| clone() | AssetEditorExportAssets | O(N) | Creates a deep copy of the packet and its contained AssetPath array. |

*N represents the number of elements in the **paths** array.*

## Integration Patterns

### Standard Usage
This packet is intended to be handled by the network layer. A packet processor or handler receives the deserialized object and dispatches it to the relevant application service.

```java
// Example of a packet handler processing the object
public void handleAssetExport(AssetEditorExportAssets packet) {
    if (packet.paths == null) {
        // No assets to export, handle as a no-op or log a warning.
        return;
    }

    AssetExportService service = context.getService(AssetExportService.class);
    service.beginExport(packet.paths);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and resend the same packet instance. Packets are cheap to create and should be treated as immutable after being passed to the network layer for serialization.
- **Cross-Thread Modification:** Do not deserialize a packet on a network thread and pass a reference to a worker thread for modification without proper synchronization or deep cloning. This is a direct path to race conditions.
- **Manual Deserialization:** Avoid calling **deserialize** directly. This is the responsibility of the protocol decoding pipeline, which correctly handles packet ID dispatching and buffer management.

## Data Pipeline
The AssetEditorExportAssets packet is a data record that flows through the network stack.

> **Ingress Flow (Client -> Server):**
> Raw TCP Socket -> Netty ByteBuf -> Protocol Decoder (identifies ID 342) -> **AssetEditorExportAssets.deserialize** -> Fully formed **AssetEditorExportAssets** object -> Packet Handler -> Asset Export Service

> **Egress Flow (Server -> Client or vice-versa):**
> Application Logic -> new **AssetEditorExportAssets(paths)** -> Protocol Encoder (**instance.serialize**) -> Netty ByteBuf -> Raw TCP Socket

