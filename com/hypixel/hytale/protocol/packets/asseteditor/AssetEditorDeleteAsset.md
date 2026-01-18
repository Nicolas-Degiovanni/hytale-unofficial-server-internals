---
description: Architectural reference for AssetEditorDeleteAsset
---

# AssetEditorDeleteAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorDeleteAsset implements Packet {
```

## Architecture & Concepts
The AssetEditorDeleteAsset class is a specialized Data Transfer Object, or Packet, within the Hytale network protocol. It serves a single, precise function: to encapsulate and transport a request to delete an asset from a remote asset editor session. This class is a fundamental component of the client-server communication layer for in-game or in-editor development tools.

It is not a service or a manager; it is a message. Its primary role is to act as a structured container for the data required to perform the deletion, ensuring that both the client and server agree on the exact format of the request. The class contains the logic for its own serialization and deserialization, converting its state to and from the raw byte stream used for network transmission.

Key architectural features include:
- **Immutability by Convention:** While the fields are public and mutable, the object is treated as immutable after creation or deserialization.
- **Self-Contained Serialization:** It implements the Packet interface, providing the necessary methods (serialize, computeSize) to be processed by the protocol's network pipeline.
- **Static Deserialization:** A static factory method, deserialize, is provided to construct an instance from a Netty ByteBuf, decoupling the network layer from the object's construction logic.
- **Request Correlation:** The presence of a *token* field suggests a mechanism for correlating this specific delete request with a corresponding response or for session validation.

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary scenarios:
    1.  **Client-Side:** Instantiated by the asset editor's user interface logic when a developer initiates a delete operation. The token and target AssetPath are populated before the packet is submitted to the network pipeline for serialization.
    2.  **Server-Side:** Instantiated by the protocol's packet deserializer when a data frame with PACKET_ID 329 is received from a client. The static deserialize method is invoked to populate the object from the incoming ByteBuf.
- **Scope:** The object's lifetime is extremely brief and transactional. It exists only for the duration of a single network operation—from its creation to its processing by a packet handler.
- **Destruction:** The object is not managed by any container or registry. It becomes eligible for garbage collection immediately after its data has been consumed by the relevant handler on the receiving end, or after it has been serialized and written to the network buffer on the sending end.

## Internal State & Concurrency
- **State:** The class holds mutable state in its public fields, *token* and *path*. It is a simple data container and performs no caching or complex state management. The state represents a single, atomic request.
- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal locking or synchronization. It is designed to be created, populated, and read within a single, well-defined thread context, such as a Netty I/O worker thread or a dedicated game logic thread.

**WARNING:** Accessing or modifying an AssetEditorDeleteAsset instance from multiple threads concurrently will result in unpredictable behavior and data corruption. It must be handled within a single-threaded execution model or protected by external synchronization.

## API Surface
The public API is primarily for use by the network protocol infrastructure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AssetEditorDeleteAsset | O(N) | Constructs and populates a new instance from a byte buffer at a given offset. N is the size of the path data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided byte buffer according to the protocol specification. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy on the wire. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural validation of the packet data within a buffer without full deserialization. |
| getId() | int | O(1) | Returns the static packet identifier (329). |

## Integration Patterns

### Standard Usage
This packet is almost never instantiated or manipulated directly by feature developers. It is processed by the underlying network protocol handlers. A server-side handler would receive the packet after deserialization.

```java
// Example of a hypothetical server-side packet handler
public class AssetEditorPacketHandler {
    public void handle(AssetEditorDeleteAsset packet) {
        // Authenticate using packet.token
        boolean isAuthenticated = assetAuthService.validate(packet.token);
        if (isAuthenticated) {
            // Retrieve the asset path and delegate to the appropriate service
            AssetPath pathToDelete = packet.path;
            assetManagementService.deleteAsset(pathToDelete);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send the same packet instance. A new AssetEditorDeleteAsset object must be created for every unique delete request.
- **Manual Deserialization:** Do not call deserialize with arbitrary offsets. This method should only be called by the trusted protocol decoder, which understands the overall packet stream structure.
- **Cross-Thread Sharing:** Do not pass an instance to another thread without a proper memory-safe handoff. For example, do not add it to a shared collection that is read by another thread without synchronization.

## Data Pipeline
The flow of data encapsulated by this packet is linear and unidirectional for a single request.

> **Client-Side Flow:**
> User Input (Delete Action) -> Asset Editor Service -> **new AssetEditorDeleteAsset()** -> Protocol Serialization Engine -> Netty Channel -> Network

> **Server-Side Flow:**
> Network -> Netty Channel -> Protocol Deserialization Engine -> **AssetEditorDeleteAsset.deserialize()** -> Packet Dispatcher -> Asset Editor Handler -> Filesystem/Database Deletion

