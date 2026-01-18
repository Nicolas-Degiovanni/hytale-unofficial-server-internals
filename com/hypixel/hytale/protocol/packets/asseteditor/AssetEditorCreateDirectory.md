---
description: Architectural reference for AssetEditorCreateDirectory
---

# AssetEditorCreateDirectory

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorCreateDirectory implements Packet {
```

## Architecture & Concepts
The AssetEditorCreateDirectory class is a network packet definition within Hytale's asset editing protocol. It functions as a highly specialized Data Transfer Object (DTO) that encapsulates a single, specific command: a request to create a new directory within the asset file system.

This packet is a fundamental component of the real-time, remote asset editing pipeline. It enables external tools, such as the Hytale Model Maker or other developer utilities, to manipulate the game's asset structure without direct file system access. The class acts as a structured message, translating a user action in a tool into a command that the game client or server can understand and execute.

Its design emphasizes network efficiency and protocol rigidity. The serialization and deserialization logic is manually implemented to ensure precise control over the byte-level representation on the wire, including optimizations like a bitmask for nullable fields.

## Lifecycle & Ownership
- **Creation:** An instance is created on the sending system (e.g., an asset editor) when a user initiates a "create directory" action. On the receiving system (the game client or server), a new instance is created exclusively by the static *deserialize* method as part of the network layer's packet processing pipeline.
- **Scope:** The object's lifetime is exceptionally brief and transactional. On the sender side, it exists only long enough to be serialized into a network buffer. On the receiver side, it exists only for the duration of its processing by a corresponding packet handler before being discarded.
- **Destruction:** The object is short-lived and becomes eligible for garbage collection immediately after its data has been used. On the sender, this occurs after serialization; on the receiver, after the packet handler has processed the command.

## Internal State & Concurrency
- **State:** The internal state is mutable and minimal, consisting only of the data required to execute the command: a *token* for request tracking and the target *path*. It does not cache any data and represents a single, atomic operation.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container intended for use within a single-threaded context, such as a Netty I/O thread or a main game loop tick. Concurrent modification from multiple threads will lead to unpredictable behavior and data corruption. All interaction should be confined to the thread that deserialized it.

## API Surface
The public contract is dominated by the static methods that define its role within the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorCreateDirectory | O(N) | **[Static]** Constructs a new packet by reading from a ByteBuf. N is the size of the path. |
| serialize(buf) | void | O(N) | Writes the packet's state into a ByteBuf for network transmission. N is the size of the path. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy when serialized. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Static]** Verifies if the data in a buffer represents a valid packet without full deserialization. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game logic. It is processed by a dedicated packet handler which extracts the data and delegates the operation to the appropriate asset management service. The *token* is used to correlate the request with a subsequent response.

```java
// Example of a packet handler processing this packet
public void handle(AssetEditorCreateDirectory packet) {
    AssetPath targetPath = packet.path;
    int requestToken = packet.token;

    // Delegate to the core asset system
    AssetSystem assetSystem = context.getService(AssetSystem.class);
    boolean success = assetSystem.createDirectory(targetPath);

    // Send a response packet back to the tool
    networkManager.send(new AssetEditorResponse(requestToken, success));
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send the same packet instance. Each network command must be represented by a new, distinct packet object to ensure transactional integrity.
- **Manual Deserialization:** Do not attempt to read the fields from the ByteBuf manually. Always use the static *deserialize* method, as it correctly handles the internal data layout, including the null-field bitmask.
- **Cross-Thread Access:** Never pass a packet instance to another thread for processing. The handler that receives the packet is responsible for extracting the data and passing primitive or immutable values to other threads if necessary.

## Data Pipeline
The class serves as a data container that moves a command from an external tool into the game engine's asset system via the network stack.

> Flow:
> User Action in Tool -> **AssetEditorCreateDirectory** (Sender) -> Serialization -> Netty ByteBuf -> Network -> Netty ByteBuf -> Deserialization -> **AssetEditorCreateDirectory** (Receiver) -> Packet Handler -> Asset System Call

