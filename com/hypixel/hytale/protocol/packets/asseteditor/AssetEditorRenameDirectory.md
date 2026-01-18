---
description: Architectural reference for AssetEditorRenameDirectory
---

# AssetEditorRenameDirectory

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorRenameDirectory implements Packet {
```

## Architecture & Concepts
The AssetEditorRenameDirectory class is a specialized Data Transfer Object, or *Packet*, within the Hytale network protocol. It does not represent a service or a manager; it is a pure data structure whose sole purpose is to convey a single, specific command from a client to a server: to rename a directory within the live-editing asset system.

This packet is part of a dedicated sub-protocol for the Asset Editor, identified by its package and unique Packet ID (309). Its design is a clear indicator of the engine's focus on performance for developer tooling. Rather than using a generic serialization format like JSON, it employs a highly optimized, custom binary layout. This layout is explicitly defined by class constants such as FIXED_BLOCK_SIZE and VARIABLE_BLOCK_START, which allows the protocol layer to perform zero-copy reads and pre-calculate buffer sizes with high precision.

The structure of the packet serialization is non-trivial. It consists of three main parts:
1.  A **Nullable Bit Field**: A single byte at the start of the payload acts as a bitmask to indicate which of the nullable fields (path, newPath) are present. This avoids wasting bytes sending null terminators or length prefixes for absent data.
2.  A **Fixed-Size Block**: Contains the token and offsets to the variable data. This block is always present and has a predictable size, allowing for fast, direct memory access.
3.  A **Variable-Size Block**: Contains the actual payload for complex types like AssetPath. The offsets in the fixed block point to the start of each data structure within this block.

This architecture minimizes network overhead and reduces CPU cycles spent on parsing, which is critical for a responsive real-time editing experience.

### Lifecycle & Ownership
-   **Creation:** An instance is created on the sending endpoint, typically the client, in response to a user action within the Asset Editor UI. For example, when a developer right-clicks a folder and chooses "Rename". The object is populated with a request token, the original path, and the new path.
-   **Scope:** The object's lifetime is extremely brief and transient. It exists only for the duration of a single network transaction: creation, serialization, network transit, deserialization, and processing. It is not intended to be stored or referenced after the initial command has been handled.
-   **Destruction:** Once the packet is deserialized on the receiving endpoint, its data is extracted and used by a corresponding packet handler. The AssetEditorRenameDirectory object itself is then immediately eligible for garbage collection. The underlying Netty ByteBuf is managed and released by the network framework.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its public fields can be modified after instantiation. However, by convention, an instance should be treated as immutable once it has been submitted to the network layer for serialization.
-   **Thread Safety:** **This class is not thread-safe.** It contains no synchronization mechanisms. It is designed to be created, populated, and serialized on a single thread. On the receiving end, it is deserialized by a Netty I/O worker thread and should be immediately passed to a single-threaded game logic handler. Any concurrent access to an instance of this class will result in undefined behavior and potential data corruption in the serialized byte stream.

## API Surface
The primary contract of this class is for serialization and deserialization, managed by the protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(N) | Encodes the object's state into a binary format in the provided Netty buffer. N is the total length of the path strings. |
| deserialize(ByteBuf buf, int offset) | AssetEditorRenameDirectory | O(N) | A static factory method that decodes binary data from a buffer into a new class instance. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(N) | Performs a low-level structural check on the raw bytes in a buffer without full deserialization. Critical for security and stability. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will occupy when serialized. Used for buffer pre-allocation. |

## Integration Patterns

### Standard Usage
This packet is never handled directly by application-level code. Instead, the network engine's dispatcher uses the packet ID to route the raw buffer to the static deserialize method. The resulting object is then passed to a registered handler for processing.

```java
// Hypothetical Packet Handler on the Server
public class AssetEditorPacketHandler {

    public void handleRenameDirectory(AssetEditorRenameDirectory packet) {
        // The 'packet' object is fully deserialized and ready for use.
        int requestToken = packet.token;
        AssetPath oldPath = packet.path;
        AssetPath newPath = packet.newPath;

        // WARNING: Perform validation and permission checks here.
        // Do not trust the data blindly.
        boolean success = assetService.renameDirectory(oldPath, newPath);

        // Send a response packet back to the client.
        sendResponse(requestToken, success);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Caching:** Do not store instances of this packet in a cache or as part of a component's state. They represent a point-in-time command, not persistent data.
-   **Manual Serialization:** Avoid calling serialize or deserialize directly. These methods are tightly coupled to the engine's protocol layer, which manages buffer allocation and packet framing. Manual invocation risks buffer leaks and data corruption.
-   **Reusing Instances:** Do not modify and resend the same packet instance. Always create a new object for each distinct command to ensure data integrity.

## Data Pipeline
The flow of data for this packet is unidirectional and represents a remote procedure call from the client's editor to the server's asset management system.

> **Client Flow:**
> User Input (UI Rename Event) -> Command Creation -> **AssetEditorRenameDirectory (Instance Created)** -> Protocol Engine (serialize) -> Netty Channel -> TCP Stream
>
> ---
>
> **Server Flow:**
> TCP Stream -> Netty Channel (read to ByteBuf) -> Protocol Engine (dispatch by ID) -> **AssetEditorRenameDirectory.deserialize** -> Packet Handler -> Asset Filesystem Logic

