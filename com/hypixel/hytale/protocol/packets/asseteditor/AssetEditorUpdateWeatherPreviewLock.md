---
description: Architectural reference for AssetEditorUpdateWeatherPreviewLock
---

# AssetEditorUpdateWeatherPreviewLock

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorUpdateWeatherPreviewLock implements Packet {
```

## Architecture & Concepts
The AssetEditorUpdateWeatherPreviewLock class is a network **Packet**, a fundamental Data Transfer Object within the Hytale protocol. It does not contain any logic; its sole purpose is to encapsulate and transport a single piece of state—a boolean flag—between a client and a server, specifically within the context of the in-game asset editor.

This packet represents a user command to lock or unlock the weather simulation in the asset preview viewport. It is part of a larger family of packets prefixed with *AssetEditor*, which collectively define the remote procedure call (RPC) interface for manipulating assets.

The static fields such as PACKET_ID, IS_COMPRESSED, and FIXED_BLOCK_SIZE are critical metadata. This metadata is consumed by the core protocol engine to perform dispatching, serialization, and deserialization without needing to inspect the packet's instance data via reflection. This design ensures high-performance network I/O by allowing the engine to pre-calculate buffer sizes and handler routes.

### Lifecycle & Ownership
- **Creation:** An instance is created on the sending endpoint (typically the client) in direct response to a user action, such as toggling a checkbox in the asset editor's UI. It is a short-lived, single-purpose object.
- **Scope:** The object's lifetime is exceptionally brief. It exists only for the duration of its serialization into a network buffer. On the receiving end, a new instance is created during deserialization and is discarded immediately after its data is consumed by a packet handler.
- **Destruction:** Managed entirely by the Java Garbage Collector. Once the packet is processed and no longer referenced by the network stack or a handler, it becomes eligible for collection.

## Internal State & Concurrency
- **State:** The class holds a single, **mutable** public field: *locked*. While technically mutable, instances of this packet should be treated as immutable after construction. The state is intended to be "write-once, read-once".
- **Thread Safety:** This class is **not thread-safe**. It is a plain data structure with no synchronization mechanisms. It is designed to be created on a primary thread (e.g., the UI thread), serialized by a network thread, and a new instance deserialized and processed on a separate I/O worker thread. Sharing a single instance across threads is an anti-pattern and will lead to race conditions.

## API Surface
The public API is primarily intended for use by the Hytale protocol framework, not for direct invocation by feature developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorUpdateWeatherPreviewLock(boolean) | constructor | O(1) | Creates a new packet with the specified lock state. |
| getId() | int | O(1) | Returns the unique, static network identifier for this packet type (354). |
| serialize(ByteBuf) | void | O(1) | Writes the boolean state as a single byte to the provided network buffer. |
| deserialize(ByteBuf, int) | static AssetEditorUpdateWeatherPreviewLock | O(1) | Reads one byte from the buffer and constructs a new packet instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-deserialization check to ensure the buffer contains sufficient data. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload in bytes (1). |

## Integration Patterns

### Standard Usage
This packet is not used directly. It is instantiated and passed to a network service or client context, which manages its serialization and transmission.

```java
// Executed on the client in response to a UI event
// The NetworkClient is a hypothetical service that handles packet dispatch
boolean isPreviewLocked = true;
Packet lockPacket = new AssetEditorUpdateWeatherPreviewLock(isPreviewLocked);
networkClient.sendPacket(lockPacket);

// On the server, a handler would be registered for this packet ID
// The handler receives the deserialized object
public void handleWeatherPreviewLock(AssetEditorUpdateWeatherPreviewLock packet) {
    this.assetEditorService.setWeatherLocked(packet.locked);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Queuing:** Do not modify the *locked* field after the packet has been passed to a network service for sending. The serialization may happen on a different thread at a later time, leading to a data race.
- **Manual Serialization:** Never call `serialize` or `deserialize` directly. These methods are strictly for the internal protocol engine. Interacting with them manually can corrupt the network stream.
- **Instance Re-use:** Do not cache and re-send packet instances. They are extremely lightweight and should be created anew for each distinct command.

## Data Pipeline
The flow of this data is unidirectional, from the command initiator (client) to the stateful authority (server or local preview engine).

> Flow:
> User UI Interaction -> Event Handler -> **new AssetEditorUpdateWeatherPreviewLock(state)** -> Protocol Engine (Serialization) -> Netty I/O Thread -> TCP/IP Stack -> Server -> Netty I/O Thread -> Protocol Engine (Deserialization) -> Packet Handler -> Asset Editor State Update

