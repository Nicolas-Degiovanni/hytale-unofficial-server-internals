---
description: Architectural reference for AssetEditorRedoChanges
---

# AssetEditorRedoChanges

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient (Packet Data Transfer Object)

## Definition
```java
// Signature
public class AssetEditorRedoChanges implements Packet {
```

## Architecture & Concepts
The AssetEditorRedoChanges class is a network packet definition within the Hytale Protocol Layer. It serves as a Data Transfer Object (DTO) specifically designed to communicate a "redo" command for the in-game asset editor. This class encapsulates the data required for a remote system—typically the server or a connected tool—to re-apply a previously undone change to a specific asset.

As an implementation of the Packet interface, it adheres to a strict contract for serialization, deserialization, and size computation, ensuring compatibility with the engine's network pipeline. The design separates the data representation of a command from its execution logic. The class itself contains no business logic; it is a pure data structure whose state is serialized for network transport and deserialized by the receiving endpoint.

The presence of a `token` field suggests a transactional or session-based system for asset modifications, allowing the server to correlate this command with a specific editing session or user. The nullable `AssetPath` field provides context, specifying which asset the redo operation targets.

## Lifecycle & Ownership
-   **Creation:** An instance of AssetEditorRedoChanges is created on the sending endpoint (e.g., the game client) in response to a user action, such as clicking a "redo" button in the asset editor UI. The constructor is populated with the current session token and the target asset path.
-   **Scope:** The object's lifetime is exceptionally short and confined to the network transaction. It exists only long enough to be serialized into a byte buffer, transmitted, and then deserialized on the receiving end.
-   **Destruction:** Once the packet is deserialized by the receiver, its data is consumed by a handler. The AssetEditorRedoChanges object itself is then dereferenced and becomes eligible for garbage collection. It is not designed for long-term storage.

## Internal State & Concurrency
-   **State:** The class holds mutable state through its public fields, `token` and `path`. While technically mutable, it is intended to be treated as immutable after its initial construction and population. The state represents a single, atomic command.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization. It must be created, populated, and passed to the network layer from a single thread. Any concurrent modification from multiple threads will lead to race conditions and unpredictable behavior during serialization. All interaction should be externally synchronized or confined to a single-threaded context like a network I/O loop or a main game thread.

## API Surface
The public API is dominated by static methods and interface implementations required by the Hytale network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (350) for this packet type. |
| serialize(ByteBuf) | void | O(N) | Serializes the object's state into the provided Netty ByteBuf. N is the size of the AssetPath. |
| deserialize(ByteBuf, int) | AssetEditorRedoChanges | O(N) | Static factory method. Deserializes data from a ByteBuf to construct a new instance. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy on the wire. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a structural integrity check on the buffer without full deserialization. Critical for security and stability. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the protocol layer. A packet handler on the receiving end identifies the packet by its ID, then uses the static `deserialize` method to construct the object from the network buffer. The resulting object is then passed to a dedicated service responsible for processing asset editor commands.

```java
// Example: In a network packet handler
void handlePacket(ByteBuf buffer) {
    // Assuming packet ID 350 was already read
    AssetEditorRedoChanges redoCmd = AssetEditorRedoChanges.deserialize(buffer, buffer.readerIndex());

    // Pass the command to the appropriate system
    AssetEditingService service = context.getService(AssetEditingService.class);
    service.processRedoCommand(redoCmd);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Queuing:** Do not modify the fields of an AssetEditorRedoChanges object after it has been submitted to the network queue for sending. This creates a race condition between the application thread and the network I/O thread.
-   **Manual Serialization/Deserialization:** Never attempt to read or write the packet's fields to a buffer manually. Always use the provided `serialize` and `deserialize` methods to ensure correctness and forward compatibility.
-   **Reusing Instances:** Packet instances are cheap to create. Do not attempt to pool or reuse them, as their internal state is not designed for reset. Create a new instance for every command.

## Data Pipeline
The flow of data for this packet follows a standard command pattern over the network.

> Flow:
> User Action (UI) -> Command Instantiation (**AssetEditorRedoChanges**) -> Network Layer (Serialization) -> TCP/IP Stack -> Network Layer (Deserialization) -> Packet Handler -> Asset Editing Service (Execution)

