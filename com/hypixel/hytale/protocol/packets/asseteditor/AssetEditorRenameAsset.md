---
description: Architectural reference for AssetEditorRenameAsset
---

# AssetEditorRenameAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AssetEditorRenameAsset implements Packet {
```

## Architecture & Concepts
The AssetEditorRenameAsset class is a Data Transfer Object (DTO) that represents a single, specific command within the Hytale network protocol. Its sole purpose is to encapsulate the data required to instruct a remote system, typically a server, to rename a version-controlled asset.

This class is a fundamental component of the client-server communication layer for the Hytale Asset Editor. It follows the Command pattern, where the object itself is the message, containing all necessary parameters for the operation. It is designed for high-performance binary serialization and deserialization, integrating directly with the Netty networking framework, as evidenced by its use of the ByteBuf type.

The presence of static metadata fields like PACKET_ID, FIXED_BLOCK_SIZE, and MAX_SIZE indicates that this packet is part of a sophisticated, custom-built protocol engine. This engine likely uses the PACKET_ID to route incoming byte streams to the correct deserializer, in this case, AssetEditorRenameAsset.deserialize. The structure is optimized to minimize network overhead by using techniques like bit fields for nullability checks and pre-calculated offsets for variable-length data.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **On the sending client:** When a user initiates a rename operation in the asset editor, the application logic instantiates this class via its constructor, populating it with the token, original path, and new path.
    2.  **On the receiving server:** The network protocol layer invokes the static `deserialize` factory method to construct an object from an incoming network ByteBuf.

- **Scope:** The object's lifetime is extremely short and confined to a single transaction. It exists only to be serialized, transmitted, deserialized, and immediately processed by a command handler. It does not persist beyond the scope of the network event that processes it.

- **Destruction:** The object is eligible for garbage collection as soon as the corresponding command handler completes its execution. It has no managed resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is a mutable data container. Its public fields—token, path, and newPath—are directly accessible and intended to be populated once during creation or deserialization. It holds no internal caches or derived state.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed by a single thread, typically a Netty I/O worker thread. Sharing an instance across multiple threads without explicit external synchronization will lead to race conditions and unpredictable behavior.

## API Surface
The public API is designed to fulfill the contract of the Packet interface and support the protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique network identifier (328) for this packet type. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into a binary format in the provided buffer. |
| deserialize(ByteBuf, int) | AssetEditorRenameAsset | O(N) | Static factory. Decodes binary data from a buffer into a new object instance. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a security-critical check on a buffer to ensure it represents a valid packet without full deserialization. |
| clone() | AssetEditorRenameAsset | O(N) | Creates a deep copy of the packet and its contained AssetPath objects. |

*Complexity O(N) is relative to the combined length of the string data within the AssetPath fields.*

## Integration Patterns

### Standard Usage
The class is used differently by the packet sender (client) and receiver (server).

**Client-Side: Sending the Command**
```java
// 1. Create the asset paths
AssetPath oldAsset = new AssetPath("models/monsters/creeper.json");
AssetPath newAsset = new AssetPath("models/monsters/creeper_v2.json");

// 2. Construct the packet with a transaction token
int transactionToken = 12345;
AssetEditorRenameAsset packet = new AssetEditorRenameAsset(transactionToken, oldAsset, newAsset);

// 3. The network layer serializes and sends it
channel.writeAndFlush(packet);
```

**Server-Side: Receiving and Processing the Command**
```java
// The protocol dispatcher has already identified the packet ID and called deserialize
public void handleRenameCommand(AssetEditorRenameAsset command) {
    // WARNING: Always validate user-controlled paths before use
    if (isValid(command.path) && isValid(command.newPath)) {
        boolean success = assetService.rename(command.path, command.newPath);
        // Respond to the client with the result using the token
        sendRenameAck(command.token, success);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Send:** Do not modify the packet's fields after it has been passed to the network layer for sending. The serialization may happen on a different thread, leading to data corruption.
- **Ignoring Validation:** Never call `deserialize` on a ByteBuf from an untrusted source without first calling `validateStructure`. Failure to do so can expose the server to denial-of-service attacks or buffer-related vulnerabilities.
- **Sharing Instances:** Do not cache or share packet instances across different network events or threads. They are cheap to create and are not designed for concurrent access.

## Data Pipeline
The AssetEditorRenameAsset packet facilitates a clear, one-way command flow from the client to the server.

> **Flow:**
> Client UI Event (User renames file) -> `new AssetEditorRenameAsset(...)` -> **Serialization Engine** -> Netty Channel -> Network -> Server Netty Channel -> **Protocol Dispatcher (reads PACKET_ID)** -> `AssetEditorRenameAsset.deserialize()` -> **Asset Command Handler** -> Filesystem/VCS Operation -> Response Packet Sent to Client

