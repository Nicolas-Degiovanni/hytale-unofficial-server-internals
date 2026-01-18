---
description: Architectural reference for AssetEditorDeleteDirectory
---

# AssetEditorDeleteDirectory

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorDeleteDirectory implements Packet {
```

## Architecture & Concepts
The AssetEditorDeleteDirectory class is a network Packet, a specialized Data Transfer Object (DTO) designed for high-performance, low-level network communication. It represents a single, immutable command within the Asset Editor protocol. Its sole purpose is to encapsulate the data required to request the deletion of a directory within the game's asset virtual file system.

This class is a fundamental component of the client-server communication pipeline for live asset editing. It is not a service or a manager; it is a transient data structure that exists only to be serialized for network transport or deserialized for server-side processing. The design prioritizes wire-format efficiency and deserialization speed over object-oriented encapsulation, a common and necessary trade-off in high-throughput networking layers.

The presence of a `token` field indicates its role in a request-response or command-acknowledgment pattern, allowing the client to correlate this specific delete request with a subsequent success or failure response from the server.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client-side (e.g., the asset editor tool) when a user initiates a directory deletion action. On the server-side, an instance is created by the protocol layer when a corresponding network buffer is received and passed to the static `deserialize` method.
- **Scope:** The object's lifetime is exceptionally short and confined to a single network transaction. On the client, it is created, serialized, and immediately becomes eligible for garbage collection. On the server, it is deserialized, processed by a handler, and then discarded.
- **Destruction:** The object is managed by the Java Garbage Collector. There is no manual cleanup or destruction logic. Its ownership is never transferred; it is processed and then abandoned.

## Internal State & Concurrency
- **State:** The internal state is fully mutable via public fields (`token`, `path`). This is an intentional design choice to facilitate high-speed, zero-overhead serialization and deserialization. The object is effectively a structured container for data and is not intended to enforce invariants on its own state after construction.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, populated, and processed within the confines of a single network thread (e.g., a Netty I/O worker thread). Any concurrent access or modification will lead to race conditions and unpredictable behavior. All synchronization must be handled externally by the calling system.

## API Surface
The public contract is focused entirely on network protocol operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorDeleteDirectory(token, path) | constructor | O(1) | Constructs a new packet instance for serialization. |
| deserialize(buf, offset) | static AssetEditorDeleteDirectory | O(N) | Constructs a packet by reading from a Netty ByteBuf. N is the size of the path. |
| serialize(buf) | void | O(N) | Writes the packet's state into a Netty ByteBuf for network transport. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will consume on the wire. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a non-deserializing check on a buffer to validate packet integrity. |
| clone() | AssetEditorDeleteDirectory | O(N) | Creates a deep copy of the packet. Use is discouraged in performance-critical paths. |

## Integration Patterns

### Standard Usage
This packet is never used directly by typical game logic. It is created by the client-side network layer and consumed by the server-side protocol handler. The handler is responsible for interpreting the packet and delegating the deletion task to the appropriate asset management service.

```java
// Client-side: Sending the request
int transactionToken = generateToken();
AssetPath directoryToDelete = new AssetPath("models/props/legacy");
AssetEditorDeleteDirectory packet = new AssetEditorDeleteDirectory(transactionToken, directoryToDelete);

// The network layer will then call packet.serialize(byteBuf) and send it.

// Server-side: Inside a protocol handler
// byteBuf is received from the network
AssetEditorDeleteDirectory request = AssetEditorDeleteDirectory.deserialize(byteBuf, 0);
AssetService service = context.getService(AssetService.class);
boolean success = service.deleteDirectory(request.path);

// A response packet would then be sent back using request.token
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and resend the same packet instance. A new packet must be created for every distinct command.
- **Cross-Thread Access:** Do not deserialize a packet on a network thread and pass the reference to a worker thread without creating a defensive, immutable copy. The raw packet object is not safe for concurrent access.
- **Manual Instantiation on Server:** Do not use `new AssetEditorDeleteDirectory()` on the server side for processing. The only valid way to create an instance on the server is via the static `deserialize` method.

## Data Pipeline
The data flow for this packet is linear and unidirectional for a single transaction.

> Flow:
> Asset Editor UI Event -> Client Network Layer creates **AssetEditorDeleteDirectory** -> `serialize()` -> Netty ByteBuf -> TCP/IP Stack -> Server Network Layer -> `deserialize()` -> **AssetEditorDeleteDirectory** instance -> Protocol Handler -> Asset Management Service -> Filesystem I/O Operation

