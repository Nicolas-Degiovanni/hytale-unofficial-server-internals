---
description: Architectural reference for AssetEditorFetchLastModifiedAssets
---

# AssetEditorFetchLastModifiedAssets

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorFetchLastModifiedAssets implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchLastModifiedAssets class is a network packet definition within the Hytale protocol. It represents a specific, zero-payload command sent from a client to a server. Its primary role is to function as a signal or a trigger, instructing the server to initiate a search for recently modified game assets.

This packet is a fundamental component of the live asset editing and hot-reloading system. When a tool like the Hytale Model Maker is connected to a running game server, it sends this packet to request an updated list of assets. The packet itself carries no data; its identity, defined by the constant PACKET_ID 338, is the entire message. The server's network layer identifies the packet by this ID and routes it to a handler responsible for asset management, which then queries the asset database and sends the results back to the client using different packet types.

## Lifecycle & Ownership
-   **Creation:** An instance is created in one of two scenarios:
    1.  On the client-side, by application code that needs to send the request (e.g., `new AssetEditorFetchLastModifiedAssets()`).
    2.  On the server-side, by the protocol's deserialization pipeline when a packet with ID 338 is read from an incoming network buffer. The static `deserialize` method is the designated factory for this process.
-   **Scope:** The object's lifetime is extremely brief. It is a transient Data Transfer Object (DTO) that exists only for the duration of a single network event processing cycle.
-   **Destruction:** The object is eligible for garbage collection immediately after being processed by the relevant network handler. No long-term references are maintained.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is defined entirely by its type. All methods return constant values or new, identical instances.
-   **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless and immutable nature, an instance can be safely shared and accessed across multiple threads without synchronization. In practice, it is typically handled by a single Netty I/O worker thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique network identifier for this packet type, which is always 338. |
| deserialize(ByteBuf, int) | AssetEditorFetchLastModifiedAssets | O(1) | Static factory method to construct an instance from a network buffer. Consumes zero bytes. |
| serialize(ByteBuf) | void | O(1) | Encodes the packet into a network buffer. This is a no-op as the packet has no payload. |
| computeSize() | int | O(1) | Calculates the serialized size of the packet, which is always 0. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct manipulation by most game logic. It is created and sent by a client-side system, and received and processed by a server-side network handler.

```java
// Client-side: Sending the request
Packet request = new AssetEditorFetchLastModifiedAssets();
networkChannel.send(request);

// Server-side: Handling the request (conceptual)
public void handlePacket(AssetEditorFetchLastModifiedAssets packet) {
    // The presence of the packet is the signal
    List<Asset> modifiedAssets = assetService.findLastModified();
    // Send results back to the client via other packet types
    networkChannel.send(new AssetEditorLastModifiedAssetsResponse(modifiedAssets));
}
```

### Anti-Patterns (Do NOT do this)
-   **Adding State:** Do not modify this class to add fields. Its contract is to be a zero-byte signal. If data needs to be sent, a new packet type must be created.
-   **Manual Codec Calls:** Never call `serialize` or `deserialize` directly in application code. These methods are exclusively for the network protocol codec, which manages the byte-level translation on the Netty pipeline.

## Data Pipeline
The data flow for this packet is a simple client-to-server request trigger.

> Flow:
> Asset Editor Client (User Action) -> **new AssetEditorFetchLastModifiedAssets()** -> Protocol Encoder -> Server Network Channel -> Protocol Decoder -> **AssetEditorFetchLastModifiedAssets Instance** -> Server Network Handler -> Asset System Query

