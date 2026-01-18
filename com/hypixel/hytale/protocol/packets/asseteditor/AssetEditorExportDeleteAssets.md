---
description: Architectural reference for AssetEditorExportDeleteAssets
---

# AssetEditorExportDeleteAssets

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorExportDeleteAssets implements Packet {
```

## Architecture & Concepts
The AssetEditorExportDeleteAssets class is a network packet definition within the Hytale protocol. It serves as a Data Transfer Object (DTO), a pure data container with no inherent logic, designed to transport information between a client and a server. Its specific purpose is to communicate a request to delete one or more assets as part of the Asset Editor's live-sync functionality.

This class is a concrete implementation of the Packet interface, making it a recognized message type within the engine's network layer. The network layer uses the static metadata, such as PACKET_ID, to identify and route incoming byte streams to the correct deserializer. The class's primary responsibility is to provide a type-safe, object-oriented representation of a low-level byte message, abstracting the complexities of network byte ordering, variable-length integers, and null-field tracking from the rest of the application.

It exists at the boundary between the high-level game logic (the Asset Editor system) and the low-level network protocol stack (built on Netty).

### Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer:** Instantiated directly by application logic when a delete operation is triggered. For example, an AssetEditorUIService would create a new AssetEditorExportDeleteAssets, populate the asset array, and submit it to the network pipeline for transmission.
    -   **Receiving Peer:** Instantiated by the protocol's deserialization layer. The network engine reads the packet ID from an incoming ByteBuf and invokes the static `deserialize` method, which constructs the object from the byte stream. It is never created with `new` on the receiving side.
-   **Scope:** Transient and extremely short-lived. An instance exists only for the duration of a single network event. On the sender, it is eligible for garbage collection immediately after serialization. On the receiver, it is eligible for garbage collection as soon as the corresponding packet handler has finished processing it.
-   **Destruction:** Managed by the Java Garbage Collector. There are no manual cleanup or `close` methods required.

## Internal State & Concurrency
-   **State:** The class holds a single mutable field: `asset`, which is a nullable array of AssetEditorAsset. The state is simple, representing a list of assets targeted for deletion. The object itself does not cache data or maintain connections to other systems.
-   **Thread Safety:** This class is **not thread-safe**. As a simple DTO, it contains no synchronization mechanisms. It is designed to be created, populated, and then passed to the network stack, or created by the network stack and passed to a single handler thread.

    **Warning:** Concurrent modification of the `asset` field during a serialization operation will result in undefined behavior, data corruption, or runtime exceptions. Packet objects should be treated as effectively immutable after being handed off to another thread or system.

## API Surface
The public API is dominated by methods required by the Packet interface and the protocol framework for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty ByteBuf according to the protocol specification. Throws ProtocolException on constraint violations (e.g., array too long). |
| deserialize(ByteBuf, int) | static AssetEditorExportDeleteAssets | O(N) | Decodes bytes from a ByteBuf into a new AssetEditorExportDeleteAssets instance. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the serialized packet will occupy. Used by the network layer for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the byte stream to ensure it represents a valid packet structure. This is a critical security and stability measure to reject malformed packets early, before full deserialization is attempted. |

*N represents the number of elements in the `asset` array.*

## Integration Patterns

### Standard Usage
This packet is used exclusively by the network layer and the systems that interact with it. A developer will typically interact with it in a packet handler.

**Sending a Packet (Conceptual):**
```java
// Executed by a high-level system, e.g., an Asset Editor Service
AssetEditorAsset[] assetsToDelete = ... // Get assets to delete
AssetEditorExportDeleteAssets packet = new AssetEditorExportDeleteAssets(assetsToDelete);
networkClient.sendPacket(packet);
```

**Receiving a Packet (Conceptual):**
```java
// A packet handler method, invoked by the network layer
public void handleAssetDelete(AssetEditorExportDeleteAssets packet) {
    if (packet.asset == null) {
        return;
    }
    for (AssetEditorAsset asset : packet.asset) {
        assetManager.scheduleForDeletion(asset.getGuid());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not modify a packet object after it has been sent or received and then attempt to reuse it. These objects are intended to be single-use. Create a new instance for each distinct message.
-   **Manual Deserialization:** Do not call the static `deserialize` method directly. The network protocol dispatcher is responsible for routing bytes to the correct packet type. Manual invocation bypasses critical pipeline steps.
-   **Ignoring Nullability:** The `asset` field is nullable. Always perform a null check before attempting to access or iterate over the array to prevent NullPointerException.

## Data Pipeline
The class acts as a structured data record that is serialized for transport and deserialized upon receipt.

> **Outbound Flow (Sending Peer):**
> Asset Editor Logic -> `new AssetEditorExportDeleteAssets()` -> Network Pipeline -> **`serialize(ByteBuf)`** -> TCP Socket

> **Inbound Flow (Receiving Peer):**
> TCP Socket -> Netty ByteBuf -> Protocol Dispatcher -> **`AssetEditorExportDeleteAssets.deserialize(ByteBuf)`** -> Packet Handler -> Asset Deletion Logic

