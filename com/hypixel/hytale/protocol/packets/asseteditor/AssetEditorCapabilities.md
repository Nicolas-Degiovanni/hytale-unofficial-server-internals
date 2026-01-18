---
description: Architectural reference for AssetEditorCapabilities
---

# AssetEditorCapabilities

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorCapabilities implements Packet {
```

## Architecture & Concepts
AssetEditorCapabilities is a Protocol Data Unit (PDU) that functions as a Data Transfer Object (DTO) within the Hytale network protocol. Its sole purpose is to communicate the set of allowed asset editing permissions from the server to a connected client. This packet is a fundamental part of the capability negotiation handshake that occurs when a client enters an asset editing session.

This class is not a service or manager; it is a pure data container. It represents a single, immutable message in the communication stream. The server constructs and sends this packet to inform the client-side UI which features to enable or disable. For example, if `canDeleteAssetPacks` is false, the client UI should render the "Delete" button as inactive.

The class design emphasizes performance and low-level network I/O, using a fixed-size binary layout for direct serialization to and from a Netty ByteBuf. The static constants like `PACKET_ID`, `FIXED_BLOCK_SIZE`, and `MAX_SIZE` are critical metadata used by the protocol's packet dispatcher to identify, validate, and route the incoming byte stream to the correct deserializer.

## Lifecycle & Ownership
- **Creation:** An instance is created on the server when a client's permissions need to be communicated. On the client, an instance is created by the static `deserialize` method within the network layer's decoding pipeline when a corresponding packet (ID 304) is received.
- **Scope:** Extremely short-lived. An instance exists only for the brief moment it is being serialized for transmission or deserialized for processing. Once its data has been consumed by a packet handler, it is eligible for garbage collection.
- **Destruction:** The object is not managed and has no explicit destruction method. It is garbage collected when no longer referenced, which is typically immediately after being processed by its designated handler on the client's network thread.

## Internal State & Concurrency
- **State:** The object's state consists of five public boolean fields. While technically mutable, instances of this class should be treated as immutable value objects once created. Modifying its state after deserialization is a severe anti-pattern.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, serialized, deserialized, and processed within a single thread, typically a Netty I/O worker thread. If the capability data needs to be accessed by other threads (e.g., the main game thread), it should be copied to a thread-safe data structure or passed via a thread-safe queue. Direct concurrent access will lead to unpredictable behavior.

## API Surface
The public contract is primarily for the network protocol framework, not for general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network ID (304) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the five boolean flags as five bytes into the provided buffer. |
| deserialize(ByteBuf, int) | AssetEditorCapabilities | O(1) | **Static Factory.** Reads five bytes from the buffer and constructs a new instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload, which is always 5 bytes. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Verifies if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This class is almost never used directly by feature developers. It is handled automatically by the network layer. A packet handler would receive the deserialized object and use it to update a client-side state model.

```java
// Example of a hypothetical Packet Handler
public class AssetEditorPacketHandler {

    private final ClientAssetEditorState editorState;

    public void handleCapabilities(AssetEditorCapabilities capabilities) {
        // The packet's data is transferred to a long-lived state manager.
        // The 'capabilities' object is now out of scope and will be GC'd.
        editorState.updatePermissions(
            capabilities.canEditAssets,
            capabilities.canCreateAssetPacks,
            capabilities.canDeleteAssetPacks
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Logic:** Do not use `new AssetEditorCapabilities()` in client-side code to check permissions. The sole source of truth is the packet received from the server.
- **State Modification:** Do not modify the fields of a received `AssetEditorCapabilities` object. Treat it as a read-only record.
- **Long-Term Storage:** Do not store a reference to this packet object. Its data should be immediately copied into a dedicated state management service or view model. Holding a reference can prevent garbage collection and implies incorrect ownership.

## Data Pipeline
The flow of this data is unidirectional from the server to the client.

> Flow:
> Server Permission Service -> **AssetEditorCapabilities (Instance A created)** -> Protocol Serializer -> TCP/IP Network -> Client Protocol Decoder -> **AssetEditorCapabilities (Instance B created)** -> Client Packet Handler -> Client UI State Model -> UI Render Update

