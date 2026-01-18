---
description: Architectural reference for AssetEditorExportAssetFinalize
---

# AssetEditorExportAssetFinalize

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorExportAssetFinalize implements Packet {
```

## Architecture & Concepts
The AssetEditorExportAssetFinalize packet is a zero-payload control message within the Hytale asset editing protocol. It functions as a **terminator** or **commit signal** in a larger sequence of network operations. Its sole purpose is to notify the remote peer that a multi-part asset export operation, likely involving a stream of preceding data packets, has been completed.

This class is a pure Data Transfer Object (DTO) with no internal state or logic. It adheres to the Packet interface, providing the necessary serialization and deserialization stubs, but these operations are no-ops as the packet carries no data. Its identity is defined entirely by its Packet ID (345).

In the context of the engine's network layer, this packet is dispatched by a central packet registry based on its ID. A corresponding handler on the receiving end is responsible for interpreting this signal and triggering the finalization logic for the asset transfer, such as reassembling chunks, saving the asset to disk, or updating database records.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Client):** Instantiated via `new AssetEditorExportAssetFinalize()` when the client-side asset export process concludes. It is then passed to the network channel for serialization and transmission.
    - **Receiving Peer (Server):** Instantiated by the protocol layer via the static `deserialize` factory method when Packet ID 345 is read from the network buffer.
- **Scope:** Extremely short-lived. An instance exists only for the duration of its handling within a single network event loop tick. It is a fire-and-forget message.
- **Destruction:** The object becomes eligible for garbage collection immediately after its corresponding handler completes execution. It is not retained or referenced by any long-term system.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields. All instances of this class are functionally identical and interchangeable. The `hashCode` method returns a constant 0.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. It can be safely created and passed between threads. However, within the Hytale network engine, packet processing is typically confined to a single Netty I/O thread per connection to prevent concurrency issues in handlers.

## API Surface
The public API is dictated by the Packet interface contract. All operations are trivial due to the packet's lack of a payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static packet identifier, 345. |
| deserialize(buf, offset) | AssetEditorExportAssetFinalize | O(1) | Static factory. Creates a new instance. Consumes 0 bytes from the buffer. |
| serialize(buf) | void | O(1) | Writes 0 bytes to the buffer. This is a no-op. |
| computeSize() | int | O(1) | Returns the serialized size of the packet, which is always 0. |
| clone() | AssetEditorExportAssetFinalize | O(1) | Creates a new, identical instance. |

## Integration Patterns

### Standard Usage
This packet is never used in isolation. It must be sent after a sequence of data-carrying packets to signal the end of a stream. The receiving handler uses the arrival of this packet as a trigger to finalize the state machine for the asset transfer.

```java
// Example: Client-side sending logic
// 1. Send all asset data chunks...
channel.write(new AssetEditorExportAssetChunk(data_part_1));
channel.write(new AssetEditorExportAssetChunk(data_part_2));

// 2. Send the finalization signal
channel.writeAndFlush(new AssetEditorExportAssetFinalize());
```

### Anti-Patterns (Do NOT do this)
- **Sending Standalone:** Sending this packet without a preceding asset data stream will likely cause an error or be ignored by the server, which will not have an active asset transfer context to finalize.
- **Adding a Payload:** Do not modify this class to carry data. If finalization requires data (e.g., a checksum), a new, distinct packet type must be created. This packet's contract is to be a zero-size signal.
- **Direct Instantiation on Receive:** On the receiving end, never use `new AssetEditorExportAssetFinalize()`. Always rely on the protocol's deserialization mechanism to create the packet object.

## Data Pipeline
This packet does not carry data itself but orchestrates the final step of a data flow. It acts as a control signal within a larger pipeline.

> Flow:
> Client Asset Editor (User Action) -> (Stream of AssetEditorExportAssetChunk packets) -> **AssetEditorExportAssetFinalize** -> Server Network Layer -> Packet Dispatcher -> AssetFinalizationHandler -> Asset Persistence Service

