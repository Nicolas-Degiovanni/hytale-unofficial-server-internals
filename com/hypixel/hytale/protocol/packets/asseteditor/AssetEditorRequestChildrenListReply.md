---
description: Architectural reference for AssetEditorRequestChildrenListReply
---

# AssetEditorRequestChildrenListReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorRequestChildrenListReply implements Packet {
```

## Architecture & Concepts
The AssetEditorRequestChildrenListReply class is a Data Transfer Object (DTO) that operates within the Hytale network protocol layer. It represents a server-to-client message, specifically designed to fulfill a request from the in-game asset editor for a list of child assets under a given directory path.

This class is not a service; it contains no business logic. Its sole responsibility is to encapsulate the data for a reply and provide the mechanisms for its own serialization and deserialization. It serves as a concrete data contract between the client's asset editor and the server's asset provider.

The design emphasizes network performance through several low-level optimizations:
- A bitmask (`nullBits`) is used to efficiently flag the presence or absence of nullable fields, minimizing payload size.
- Variable-length integers (VarInt) are used for array and string lengths, ensuring that smaller numbers use fewer bytes.
- A two-pass serialization scheme is employed, where a fixed-size header block contains offsets to a variable-size data block. This allows for efficient parsing and validation without reading the entire packet into intermediate memory structures.

## Lifecycle & Ownership
- **Creation:**
    - **Sending (Server-side):** An instance is created via its constructor (`new AssetEditorRequestChildrenListReply(...)`) by the server-side logic responsible for handling asset queries. This happens after the server has successfully retrieved the list of child asset identifiers for a requested path.
    - **Receiving (Client-side):** An instance is materialized by the network protocol layer, which invokes the static `deserialize` method. This is triggered when a raw byte buffer with the corresponding PACKET ID (322) is read from the network channel.

- **Scope:** This object is extremely short-lived. It exists only to be passed from the application logic to the network serializer, or from the network deserializer to the packet handler. It is a message, not a persistent entity.

- **Destruction:** The object becomes eligible for garbage collection immediately after its contents have been processed by the receiving-side handler (e.g., after the asset editor UI is updated). It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The class holds mutable state through its public fields, `path` and `childrenIds`. The state represents the asset directory being described and the list of its immediate children. While technically mutable, instances should be treated as immutable after creation (on the sending side) or after deserialization (on the receiving side).

- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization. It is designed to be created, populated, serialized, deserialized, and read within a single-threaded context, such as a Netty I/O worker thread or a dedicated game logic thread. Concurrent modification from multiple threads will result in data corruption and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (322). |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided byte buffer for network transmission. N is the total size of the string data. |
| deserialize(ByteBuf, int) | AssetEditorRequestChildrenListReply | O(N) | A static factory method that decodes a byte buffer into a new instance of the class. |
| computeSize() | int | O(N) | Calculates the total number of bytes this packet will occupy when serialized. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a structural integrity check on a raw buffer without full deserialization. Crucial for security and stability. |
| clone() | AssetEditorRequestChildrenListReply | O(N) | Creates a deep copy of the packet and its contained data. |

## Integration Patterns

### Standard Usage
This packet is processed by a client-side handler that listens for incoming network messages. The handler extracts the data and forwards it to the relevant system, typically the UI logic for the asset editor.

```java
// Example of a client-side packet handler
public void handlePacket(AssetEditorRequestChildrenListReply reply) {
    if (reply.path == null || reply.childrenIds == null) {
        // Handle error case: server sent an incomplete reply
        log.warn("Received an incomplete AssetEditorRequestChildrenListReply");
        return;
    }

    // Get the UI component responsible for displaying the asset tree
    AssetEditorUI assetEditor = UIManager.getComponent(AssetEditorUI.class);

    // Update the UI with the new list of children for the given path
    assetEditor.updateChildrenForPath(reply.path, reply.childrenIds);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Serialization:** Do not modify the fields of a packet instance after it has been submitted to the network channel for sending. The serialization may occur on a different thread, leading to a race condition where a partially modified object is sent.
- **Instance Reuse:** Do not cache and reuse packet instances. They are lightweight objects designed for single use. Always create a new instance for each message to ensure thread safety and prevent state leakage.
- **Ignoring Validation Results:** On the receiving end, do not ignore the result of `validateStructure` for data from untrusted sources. A malformed packet can cause `deserialize` to throw an exception or, in worse cases, lead to buffer overflows.

## Data Pipeline
This packet is a reply in a request-response flow. The data originates from a server-side asset management system and is consumed by the client-side asset editor UI.

> Flow:
> Server Asset System -> **AssetEditorRequestChildrenListReply (Instantiation)** -> Protocol Serializer -> Netty Channel -> Network -> Client Netty Channel -> Protocol Deserializer -> **AssetEditorRequestChildrenListReply (Instance)** -> Client Packet Handler -> Asset Editor UI Update

