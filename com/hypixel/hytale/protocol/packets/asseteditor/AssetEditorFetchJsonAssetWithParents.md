---
description: Architectural reference for AssetEditorFetchJsonAssetWithParents
---

# AssetEditorFetchJsonAssetWithParents

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class AssetEditorFetchJsonAssetWithParents implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchJsonAssetWithParents class is a network packet, a specialized Data Transfer Object (DTO) used exclusively within the Hytale protocol layer. It represents a client-to-server request, specifically for fetching the contents of a JSON-based asset along with its parent assets.

This packet is a fundamental component of the in-game asset editor's functionality. When a developer or user interacts with an asset in the editor, this packet is dispatched to the server to retrieve the necessary data for rendering and modification. The inclusion of "WithParents" signifies that the request is for a full hierarchical load, ensuring that all dependencies required to correctly interpret the target asset are also retrieved.

It operates within a simple request-response pattern. The client sends this packet, and the server is expected to reply with one or more packets containing the requested asset data. The *token* field serves as a correlation identifier to match the server's response to this specific request.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client when a user action necessitates loading an asset (e.g., selecting an asset from a tree view). On the server, an instance is created by the protocol's deserialization logic when an incoming network buffer is decoded.
- **Scope:** The object's lifetime is extremely short and confined to a single transaction. On the client, it exists only long enough to be serialized and written to the network channel. On the server, it exists only for the duration of its processing by a packet handler.
- **Destruction:** The object is immediately eligible for garbage collection after its use. On the client, this occurs after serialization. On the server, this occurs after the corresponding packet handler has extracted its data and initiated the asset loading process.

## Internal State & Concurrency
- **State:** The class is a mutable container for the request data. Its public fields (token, path, isFromOpenedTab) are directly accessible and are populated either by the client-side logic initiating the request or by the server-side deserializer. It holds no cached data or complex state.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed on a single network thread (e.g., a Netty event loop). It must not be shared across threads or modified concurrently without explicit, external synchronization. Doing so will lead to race conditions and unpredictable network behavior.

## API Surface
The primary contract is defined by the Packet interface and the static serialization helpers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorFetchJsonAssetWithParents | O(N) | Constructs a new packet by reading from a ByteBuf. N is the size of the asset path. |
| serialize(buf) | void | O(N) | Writes the packet's state into a ByteBuf for network transmission. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the packet. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check to ensure the buffer contains a structurally valid packet. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (311). |

## Integration Patterns

### Standard Usage
This packet is not intended for direct manipulation by most game logic. It is constructed by a client-side service responsible for asset editor operations and passed to the network layer for dispatch.

```java
// Client-side service creating and sending the request
AssetPath targetAsset = new AssetPath("hytale", "models/creature/player.json");
int requestToken = tokenProvider.nextToken();

// The packet is a simple data container for the request
AssetEditorFetchJsonAssetWithParents request = new AssetEditorFetchJsonAssetWithParents(
    requestToken,
    targetAsset,
    true
);

// The network system handles serialization and transmission
clientConnection.sendPacket(request);
```

### Anti-Patterns (Do NOT do this)
- **Packet Reuse:** Do not attempt to reuse or cache packet instances. They are cheap to allocate and their mutable state makes reuse a significant source of bugs. Always create a new instance for each request.
- **Asynchronous Modification:** Do not modify a packet's fields after it has been submitted to the network layer. The serialization may happen on a different thread, leading to a race condition.
- **Server-Side Instantiation:** Do not manually instantiate this class on the server using its constructor. Server-side instances should only ever be created by the `deserialize` method as part of the network pipeline.

## Data Pipeline
This packet represents the start of a data request flow. The primary flow describes the journey of the packet on the receiving end (the server).

> Flow:
> Network ByteBuf -> Protocol Deserializer -> **AssetEditorFetchJsonAssetWithParents (Instance)** -> Packet Dispatcher -> AssetEditorPacketHandler -> Asset Loading System -> File System / Database Read

