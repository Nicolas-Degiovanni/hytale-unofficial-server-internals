---
description: Architectural reference for AssetEditorModifiedAssetsCount
---

# AssetEditorModifiedAssetsCount

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorModifiedAssetsCount implements Packet {
```

## Architecture & Concepts
The AssetEditorModifiedAssetsCount class is a network packet, a fundamental unit of data communication within the Hytale protocol. Its sole purpose is to transmit a single integer representing the number of assets that have been modified within the asset editor. This packet is a highly specialized, fixed-size message, optimized for performance and low overhead.

As an implementation of the Packet interface, this class is a first-class citizen in the engine's networking layer. It is identified by the static constant PACKET_ID (340), which allows the central Packet Dispatcher to route the incoming byte stream to the correct deserialization logic. The presence of constants like FIXED_BLOCK_SIZE and MAX_SIZE indicates its role within a strictly defined, non-flexible binary protocol where performance and predictability are paramount.

This packet is exclusively used in the context of the asset editor, facilitating real-time communication between the editor tool and the game client, likely to signal the need for a refresh or to provide progress updates during a synchronization process.

### Lifecycle & Ownership
- **Creation:** An instance is created by the sending endpoint (e.g., the asset editor server) when it needs to inform the client about asset changes. It is a transient object, instantiated with the specific count of modified assets. Example: `new AssetEditorModifiedAssetsCount(10)`.
- **Scope:** The object's lifetime is extremely brief. It exists only for the duration of serialization, network transmission, deserialization, and handling. It is a "fire-and-forget" data container.
- **Destruction:** Once the packet is received and its data is consumed by a packet handler, the object is no longer referenced and becomes eligible for garbage collection. The underlying Netty ByteBuf it is read from or written to is managed by the network framework's own lifecycle and memory pooling systems.

## Internal State & Concurrency
- **State:** The class holds a minimal, **mutable** state consisting of a single public integer field: count. While the field is public and mutable, the object is treated as immutable after its initial construction. It serves as a pure data carrier.
- **Thread Safety:** This class is **not thread-safe**. Direct, concurrent modification of the public count field will lead to race conditions. This is not a design flaw, as the Hytale networking protocol mandates that packet processing occurs on a single, dedicated network thread (e.g., a Netty event loop). All interaction with a packet instance should be confined to its designated handler thread.

**WARNING:** Modifying a packet's state after it has been queued for transmission is a severe anti-pattern and will result in unpredictable network behavior.

## API Surface
The public API is designed for interaction with the protocol framework, not for general-purpose use by game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static packet identifier (340). |
| serialize(ByteBuf) | void | O(1) | Writes the 4-byte integer count into the provided buffer. |
| deserialize(ByteBuf, int) | AssetEditorModifiedAssetsCount | O(1) | A static factory method that constructs a new packet by reading 4 bytes from the buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the packet's payload in bytes (4). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is not intended to be used directly by high-level game systems. It is created by a sending service and consumed by a registered packet handler on the receiving end.

```java
// Sender-side logic (e.g., in the Asset Editor service)
int modifiedCount = assetRepository.getModifiedCount();
AssetEditorModifiedAssetsCount packet = new AssetEditorModifiedAssetsCount(modifiedCount);
networkManager.sendPacket(packet);

// Receiver-side logic (within a Packet Handler)
public void handle(AssetEditorModifiedAssetsCount packet) {
    int count = packet.count;
    // Do not hold a reference to the packet object itself.
    uiNotificationService.show(count + " assets have been updated.");
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Queuing:** Never modify a packet object after it has been passed to the network manager for sending. The serialization may happen on a different thread at a later time.
- **Instance Re-use:** Do not cache and re-use packet instances. They are cheap to create and are not designed for pooling or re-use. Create a new instance for every message.
- **Manual Serialization/Deserialization:** Avoid calling serialize or deserialize directly. The network protocol's dispatcher is responsible for invoking these methods at the correct point in the pipeline. Bypassing the framework can lead to buffer leaks and corruption.

## Data Pipeline
The flow of data for this packet is linear and unidirectional, from the sender's application logic to the receiver's application logic.

> **Outbound Flow (Sender):**
> Asset Editor Service -> `new AssetEditorModifiedAssetsCount(count)` -> Network Manager -> **Packet.serialize()** -> Netty Channel Pipeline -> TCP Socket

> **Inbound Flow (Receiver):**
> TCP Socket -> Netty Channel Pipeline -> Packet Dispatcher (reads ID 340) -> **AssetEditorModifiedAssetsCount.deserialize()** -> Packet Handler -> UI Notification Service

