---
description: Architectural reference for AssetEditorSubscribeModifiedAssetsChanges
---

# AssetEditorSubscribeModifiedAssetsChanges

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorSubscribeModifiedAssetsChanges implements Packet {
```

## Architecture & Concepts
The AssetEditorSubscribeModifiedAssetsChanges packet is a specialized control message within the Hytale network protocol. It serves a single, critical function: to manage a client's subscription to real-time asset modification events from the server. This is a core component of the live asset editing and hot-reloading developer workflow.

This class acts as a simple command, not a data carrier. When a client, typically the Hytale Asset Editor, sends this packet, it is instructing the server to either begin or cease broadcasting notifications about changes to game assets. This allows for an efficient, on-demand data stream, preventing the server from sending unnecessary asset data to clients that are not actively editing.

It is one of the simplest packets in the protocol, consisting of a single boolean flag. Its fixed size and minimal processing overhead make it a lightweight and efficient mechanism for toggling a server-side state machine for a specific client connection.

### Lifecycle & Ownership
- **Creation:** An instance is created on the client when a user action triggers a change in subscription status (e.g., toggling a "Live Reload" feature in the UI). On the server, an instance is created by the network protocol's deserialization pipeline when an incoming packet with ID 341 is received.
- **Scope:** Extremely short-lived. On the client, it exists only long enough to be serialized and placed into the network send buffer. On the server, it exists only for the duration of the packet handling logic before being discarded. It is a transient object.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been processed. On the client, this occurs after serialization; on the server, after the client's subscription state has been updated.

## Internal State & Concurrency
- **State:** The internal state consists of a single mutable boolean field, *subscribe*. However, by convention and design, this packet should be treated as immutable after instantiation. The static fields defining the packet structure (PACKET_ID, FIXED_BLOCK_SIZE, etc.) are constants.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal synchronization. The underlying network framework, Netty, guarantees that the serialization and deserialization of a single packet occur within a single network I/O thread, mitigating concurrency risks at the packet level. Any service that consumes this packet must handle its own thread safety when modifying shared server state (e.g., the master subscription list).

## API Surface
The primary public API is the constructor. Other public methods are part of the internal framework contract for the Packet interface and are not intended for direct invocation by application-level code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorSubscribeModifiedAssetsChanges(boolean) | constructor | O(1) | Creates a command to subscribe or unsubscribe from asset change events. |
| serialize(ByteBuf) | void | O(1) | **Framework Internal.** Encodes the packet into a byte buffer for network transmission. |
| deserialize(ByteBuf, int) | static AssetEditorSubscribeModifiedAssetsChanges | O(1) | **Framework Internal.** Decodes a byte buffer into a packet instance. |
| getId() | int | O(1) | Returns the unique network identifier for this packet type, which is 341. |

## Integration Patterns

### Standard Usage
This packet is not used directly but is created and dispatched through a higher-level network service or client connection manager.

```java
// Client-side code within an Asset Editor service
// Assume 'connection' is an object managing the server connection

public void setLiveAssetReload(boolean enabled) {
    // Create the command packet
    AssetEditorSubscribeModifiedAssetsChanges packet = new AssetEditorSubscribeModifiedAssetsChanges(enabled);

    // Dispatch the packet to the network layer for sending
    connection.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Serialization:** Never call the serialize or deserialize methods directly. This is the responsibility of the network protocol handler. Manually managing byte buffers can lead to protocol corruption and connection errors.
- **State Modification After Queuing:** Do not modify the *subscribe* field after the packet has been passed to the network service for sending. This can lead to a race condition where the state changes after serialization has already begun. Treat the object as immutable after creation.
- **Object Re-use:** Do not cache and re-use instances of this packet. The performance gain is negligible and it violates the transient, "fire-and-forget" nature of network packets, potentially leading to bugs.

## Data Pipeline
The flow of this command is unidirectional from the client to the server.

> **Flow:**
> Client UI Event (e.g., Checkbox Toggle) -> Application Logic -> **new AssetEditorSubscribeModifiedAssetsChanges()** -> Network Service -> Netty Encoder -> TCP Socket -> Server Network Listener -> Netty Decoder -> Packet Dispatcher -> Subscription Service Logic -> Server updates client's subscription state.

