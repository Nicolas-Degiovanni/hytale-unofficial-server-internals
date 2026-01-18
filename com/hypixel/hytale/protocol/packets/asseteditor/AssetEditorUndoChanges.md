---
description: Architectural reference for AssetEditorUndoChanges
---

# AssetEditorUndoChanges

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorUndoChanges implements Packet {
```

## Architecture & Concepts
The AssetEditorUndoChanges class is a Data Transfer Object (DTO) that represents a specific network command within the Hytale protocol. It is not a service or a manager; it is a simple, inert data structure whose sole purpose is to carry information from one system to another over the network.

This packet is a fundamental component of the real-time asset editing feature. It encapsulates a user's request to revert the last change made to an asset. The packet contains a transaction token for state tracking and an optional asset path, allowing for both global and granular undo operations. Its design prioritizes network efficiency and serialization performance, using a custom binary format managed by the Packet interface contract.

## Lifecycle & Ownership
- **Creation:** An instance of AssetEditorUndoChanges is created under two distinct circumstances:
    1. **On the sending endpoint (Client):** The asset editor's user interface or command system instantiates this object when a user triggers an undo action. The fields are populated with the relevant session token and asset path before serialization.
    2. **On the receiving endpoint (Server/Client):** The protocol's deserialization layer instantiates the object by calling the static `deserialize` method, populating it with data read directly from a network ByteBuf.

- **Scope:** The object's lifetime is extremely short and confined to a single transaction. It exists only for the brief moment it is being serialized, transmitted, deserialized, and processed by a packet handler.

- **Destruction:** The object is eligible for garbage collection immediately after the corresponding packet handler has finished processing its data. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is a mutable container for its data payload (token, path). Its public fields are directly accessible, which is a deliberate design choice to simplify the high-performance serialization and deserialization logic. Once deserialized on the receiving end, it should be treated as an immutable value object.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, processed, and discarded within the confines of a single network thread (e.g., a Netty event loop). Sharing an instance of this packet across multiple threads will lead to race conditions and unpredictable behavior. All interactions must be synchronized externally if multi-threaded access is unavoidable.

## API Surface
The public API is dominated by the Packet interface contract and static utility methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorUndoChanges | O(N) | **Static Factory.** Constructs a new instance from a binary buffer. N is the size of the path. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided binary buffer. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Verifies if the data in a buffer represents a valid packet without full deserialization. |
| clone() | AssetEditorUndoChanges | O(N) | Creates a deep copy of the packet and its contained data. |

## Integration Patterns

### Standard Usage
This packet is intended to be processed by a dedicated packet handler. The handler extracts the data and dispatches the undo logic to a stateful service, such as an AssetCommandManager or a SessionState service. The packet itself contains no business logic.

```java
// Example of a packet handler processing the request
public void handle(AssetEditorUndoChanges packet) {
    // Retrieve the appropriate service for handling asset commands
    AssetCommandManager commandManager = context.getService(AssetCommandManager.class);

    // Use the token to identify the correct command history
    // The path specifies which part of the asset to undo
    commandManager.undoLastAction(packet.token, packet.path);
}
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not store a reference to this packet after it has been handled. It is transient and should be discarded immediately to avoid memory leaks.
- **Manual Serialization/Deserialization:** Avoid calling `serialize` or `deserialize` directly. These methods are strictly for use by the core networking layer. Manually manipulating the byte stream is error-prone and bypasses protocol-level features like compression and validation.
- **Cross-Thread Modification:** Never modify the packet's fields from a different thread than the one that received it. If the data is needed elsewhere, copy its values into a thread-safe data structure.

## Data Pipeline
The AssetEditorUndoChanges packet is a message that flows through the network stack.

> **Outbound Flow (Client -> Server):**
> User Input (Undo Button) -> UI Command -> **AssetEditorUndoChanges (new)** -> Packet Serializer -> Netty Channel -> Raw TCP/UDP Packet

> **Inbound Flow (Server <- Client):**
> Raw TCP/UDP Packet -> Netty Channel -> Packet Deserializer -> **AssetEditorUndoChanges (new)** -> Packet Handler -> AssetCommandManager

