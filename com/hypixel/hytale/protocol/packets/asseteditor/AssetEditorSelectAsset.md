---
description: Architectural reference for AssetEditorSelectAsset
---

# AssetEditorSelectAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorSelectAsset implements Packet {
```

## Architecture & Concepts
The AssetEditorSelectAsset class is a network packet definition within Hytale's asset editing protocol. It serves as a specialized Data Transfer Object (DTO) designed to communicate a single, specific user action: the selection of an asset within a development tool or editor.

This packet is not a service or a manager; it is a pure data container. Its primary role is to encapsulate the identity of a selected asset, represented by an AssetPath, for transmission between a client (e.g., Hytale Model Maker) and a server (e.g., a live-preview game instance). It is a fundamental component of the real-time feedback loop for creative tools, enabling the server to react to changes in the editor's focus.

The serialization and deserialization logic is highly optimized for network performance. It employs a bitmask (`nullBits`) to efficiently encode the presence of its nullable fields, minimizing payload size.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Client-Side (Serialization):** Instantiated by a client-side system, such as a UI event handler, in response to a user selecting an asset. The constructor is called with the relevant AssetPath.
    2. **Server-Side (Deserialization):** Instantiated by the network protocol layer via the static `deserialize` factory method when an incoming network buffer containing this packet's ID is processed.

- **Scope:** The object's lifetime is extremely short and tied to a single network transaction. It is a classic transient object. On the client, it exists only long enough to be serialized into a Netty ByteBuf. On the server, it exists only for the duration of its processing by a corresponding packet handler.

- **Destruction:** The object is eligible for garbage collection immediately after its use case is fulfilled (serialization on the client, processing on the server). It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class holds a single, mutable field: `path` of type AssetPath. This field is nullable, and its presence is encoded into the serialized byte stream. The state of this object is a direct representation of the data being sent over the wire.

- **Thread Safety:** This class is **not thread-safe**. It is a simple, mutable container with a public field. It is designed to be created, populated, and processed by a single thread within the context of a network event loop (e.g., a Netty worker thread).

    **WARNING:** Modifying an AssetEditorSelectAsset instance from multiple threads will lead to race conditions and unpredictable serialization results. Do not share instances between threads.

## API Surface
The public contract is dominated by the Packet interface and static utility methods for the protocol framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorSelectAsset(AssetPath) | constructor | O(1) | Creates a new packet instance for serialization. |
| serialize(ByteBuf) | void | O(N) | Writes the packet's state to the given buffer. N is the size of the AssetPath. |
| deserialize(ByteBuf, int) | static AssetEditorSelectAsset | O(N) | Reads from a buffer at an offset and constructs a new packet instance. |
| computeSize() | int | O(N) | Calculates the number of bytes this packet will occupy when serialized. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Verifies if the data in the buffer at the offset represents a valid packet. |

## Integration Patterns

### Standard Usage
This packet is handled by the network layer. On the receiving end, a packet handler retrieves the data and acts upon it.

```java
// Example of a server-side packet handler
public void handleAssetSelection(AssetEditorSelectAsset packet) {
    AssetPath selectedPath = packet.path;

    if (selectedPath != null) {
        // Log the selection or trigger a system to load the asset
        // for live preview.
        AssetService assetService = context.getService(AssetService.class);
        assetService.focusOnAsset(selectedPath);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold references to this packet after it has been processed. Its purpose is fulfilled once its data is extracted. Reusing instances is unsafe.
- **Manual Deserialization:** Do not attempt to parse the ByteBuf manually. Always use the static `deserialize` method, which correctly handles the null-field bitmask and delegates to the child object's deserialization logic.
- **Off-Thread Modification:** Do not create the packet on one thread and pass it to the network thread for serialization without proper synchronization. The recommended pattern is to create and serialize the packet on the same thread.

## Data Pipeline
The flow of this data is unidirectional from the editor client to the game server.

> Flow:
> User selects asset in Editor UI -> UI Event Handler -> **new AssetEditorSelectAsset(path)** -> Packet Serializer -> Netty Channel -> Network -> Server Packet Decoder -> **AssetEditorSelectAsset.deserialize(buffer)** -> Packet Handler -> Asset Management System

