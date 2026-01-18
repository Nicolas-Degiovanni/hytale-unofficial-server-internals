---
description: Architectural reference for AssetEditorDeleteAssetPack
---

# AssetEditorDeleteAssetPack

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class AssetEditorDeleteAssetPack implements Packet {
```

## Architecture & Concepts
The AssetEditorDeleteAssetPack class is a network **Packet**, a specialized Data Transfer Object (DTO) designed for network communication. Its sole purpose is to encapsulate and transport a command to delete a specific asset pack from a remote system, identified by its unique string ID.

This packet operates within the Hytale Protocol Framework, specifically for the Asset Editor toolchain. It is not part of the core gameplay loop. As a command-style packet, it represents a single, discrete, and idempotent action. The class itself contains no business logic; its responsibilities are strictly limited to data representation and adherence to the binary serialization format defined by the protocol.

The structure is defined by its static constants and serialization methods, which dictate how the object's state is translated to and from a Netty ByteBuf. This includes metadata for the protocol layer, such as the unique PACKET_ID, and rules for handling nullable fields using a bitmask.

### Lifecycle & Ownership
- **Creation:** An instance is created on the client-side when a user action, such as clicking a delete button in the Asset Editor UI, necessitates sending a deletion command. On the server-side, a new instance is created by the protocol's deserialization pipeline when an incoming data stream matching this packet's ID is processed.
- **Scope:** The object's lifetime is extremely brief and tied to a single network operation. On the sending side, it exists only long enough to be serialized into a byte buffer by the network channel. On the receiving side, it exists from the moment of deserialization until the corresponding packet handler has finished processing the command.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection immediately after its use in the network pipeline or handler, requiring no manual resource management.

## Internal State & Concurrency
- **State:** The internal state is **mutable** and consists of a single nullable String field: *id*. While technically mutable, instances of this packet should be treated as immutable after creation. The state is minimal, representing only the necessary data for the delete command.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization. It is designed to be created, serialized, and handled within a single thread, typically a Netty I/O worker thread.

**WARNING:** Accessing or modifying an AssetEditorDeleteAssetPack instance from multiple threads will result in unpredictable behavior and data corruption. All interaction should be confined to the network event loop or passed between threads using thread-safe queues.

## API Surface
The public API is dominated by methods related to the Hytale Protocol contract for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(N) | Encodes the packet's state into the provided ByteBuf according to the protocol specification. N is the length of the ID string. |
| deserialize(ByteBuf buf, int offset) | static AssetEditorDeleteAssetPack | O(N) | Constructs a new packet instance by decoding data from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the current state of the packet. Used for buffer pre-allocation. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(N) | Performs a low-cost validation of the byte stream to ensure it represents a structurally valid packet without full deserialization. Critical for network security and stability. |

## Integration Patterns

### Standard Usage
This packet is not intended to be used directly by most game logic. Its creation is typically abstracted away by a service that communicates with the server. On the receiving end, it is processed by a registered packet handler.

**Sending a Packet (Conceptual)**
```java
// A network service or connection manager would handle the actual sending.
// A developer triggers the action, the service creates and sends the packet.
AssetEditorService editorApi = context.getService(AssetEditorService.class);
editorApi.requestDeleteAssetPack("unique-asset-pack-id-123");

// Inside the service, a new packet is created and sent.
// public void requestDeleteAssetPack(String assetId) {
//    connection.send(new AssetEditorDeleteAssetPack(assetId));
// }
```

**Handling a Packet (Conceptual)**
```java
// A packet handler class is registered to listen for this specific packet type.
public class AssetEditorPacketHandler implements PacketHandler<AssetEditorDeleteAssetPack> {
    @Override
    public void handle(AssetEditorDeleteAssetPack packet) {
        AssetRepository repository = context.getService(AssetRepository.class);
        if (packet.id != null) {
            repository.deletePackById(packet.id);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not modify and re-send the same packet instance. Each command sent over the network must be a new, distinct object to prevent state corruption and side effects.
- **Manual Serialization:** Avoid calling serialize or deserialize directly in application code. These methods are strictly for use by the low-level network protocol layer. Interacting with them manually can bypass critical pipeline steps like encryption and compression.
- **Ignoring Nullability:** The *id* field is nullable. Handlers must always check for null before attempting to use the value to prevent NullPointerException.

## Data Pipeline
The flow of this packet represents a one-way command from a client (the Asset Editor) to a server or remote tool.

> Flow:
> User Action in Asset Editor -> Command Invocation -> **new AssetEditorDeleteAssetPack("id")** -> Network Channel Pipeline (Serialization) -> TCP/IP Stream -> [REMOTE END] -> Network Channel Pipeline (Deserialization) -> Packet Handler -> Asset Management System

