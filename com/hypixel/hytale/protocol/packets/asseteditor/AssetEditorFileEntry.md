---
description: Architectural reference for AssetEditorFileEntry
---

# AssetEditorFileEntry

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorFileEntry {
```

## Architecture & Concepts
The AssetEditorFileEntry class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol, specifically designed for the real-time Asset Editor subsystem. Its sole purpose is to represent a single file or directory in a remote file system listing, such as when a client requests the asset structure from a server.

This class is not a service or a manager; it is a pure data container. Its design is heavily optimized for network performance, employing a custom binary serialization format to minimize payload size. Key characteristics of this format include:

*   **Bitmask for Nullable Fields:** A single byte (`nullBits`) at the start of the serialized block acts as a bitmask to indicate the presence or absence of optional fields, like the *path*. This avoids the need to write null terminators or length prefixes for empty data.
*   **Variable-Length Integers:** The length of the *path* string is encoded using the VarInt format, which uses fewer bytes for smaller numbers, a common optimization for network protocols.
*   **Fixed-Size Blocks:** The core data, such as the `isDirectory` boolean, is part of a fixed-size block for predictable and fast deserialization.

This object is a building block, typically used in collections within larger packets (e.g., an `AssetEditorFileListPacket`) to transfer an entire directory listing in a single network transaction.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand. They are never managed by a dependency injection container or a central registry.
    - **Inbound:** Instantiated by a higher-level packet's `deserialize` method when reading from a Netty ByteBuf.
    - **Outbound:** Instantiated directly by application logic that needs to construct a packet for transmission.
- **Scope:** The lifecycle of an AssetEditorFileEntry is extremely short and tied to the scope of the network packet it belongs to. It is created, populated, serialized (or deserialized, read), and then immediately becomes eligible for garbage collection.
- **Destruction:** Cleanup is handled exclusively by the Java Garbage Collector. There are no native resources or explicit `close` methods.

## Internal State & Concurrency
- **State:** The object's state is fully mutable. It is a simple container for a `path` and a boolean `isDirectory` flag. It does not cache any data or maintain any complex internal state.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created and accessed within a single thread, typically a Netty I/O worker thread during packet processing. Sharing an instance across multiple threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, enforcing a clear contract for network I/O.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorFileEntry | O(N) | Constructs an instance by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the path length. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a lightweight check on a buffer to see if it contains a structurally valid entry, without full deserialization. |

## Integration Patterns

### Standard Usage
AssetEditorFileEntry is almost never used in isolation. It is constructed and added to a parent packet before being sent over the network.

```java
// Example: Preparing a packet to send to the client
AssetEditorFileListPacket fileListPacket = new AssetEditorFileListPacket();

// Create entries and add them to the parent packet's list
AssetEditorFileEntry dirEntry = new AssetEditorFileEntry("models/monsters/", true);
AssetEditorFileEntry fileEntry = new AssetEditorFileEntry("models/monsters/creeper.json", false);

fileListPacket.getEntries().add(dirEntry);
fileListPacket.getEntries().add(fileEntry);

// The network layer will later serialize the packet, which in turn
// calls serialize() on each AssetEditorFileEntry.
networkChannel.writeAndFlush(fileListPacket);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and reuse an AssetEditorFileEntry instance after it has been added to a packet. These objects are cheap to create and should be treated as immutable once constructed for a specific operation.
- **Multi-threaded Access:** Never share an instance between threads. If data needs to be passed to another thread, create a deep copy or a new, distinct DTO.
- **Manual Serialization:** Do not attempt to manually write the fields to a buffer. The binary format is precise, and any deviation will cause deserialization failures. Always use the provided `serialize` and `deserialize` methods.

## Data Pipeline
The AssetEditorFileEntry acts as a data record that is serialized for transit and deserialized upon receipt. It is a passive participant in the data flow.

> Flow (Server sending a file list to Client):
> Server-side File System Scan -> **AssetEditorFileEntry instances created** -> Added to `AssetEditorFileListPacket` -> Packet Serializer calls `serialize()` -> Raw bytes written to Netty ByteBuf -> Network Transmission -> Client-side Netty ByteBuf -> Packet Deserializer calls `deserialize()` -> **New AssetEditorFileEntry instances created** -> Client UI renders the file list

