---
description: Architectural reference for AssetEditorRequestDataset
---

# AssetEditorRequestDataset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorRequestDataset implements Packet {
```

## Architecture & Concepts
The AssetEditorRequestDataset class is a specialized network packet that functions as a Data Transfer Object (DTO). Its sole purpose is to encapsulate a client's request to a server for a specific named dataset within the Asset Editor ecosystem. It represents the "question" in a simple request-response pattern, where the server is expected to reply with the corresponding data or an error.

This packet is a fundamental component of the client-server protocol for live asset editing. It is designed for low-level network I/O, containing precise serialization and deserialization logic that interacts directly with Netty byte buffers. The class itself holds no logic beyond data representation and the mechanics of its own network transport format.

Key architectural characteristics include:
- **Immutability by Convention:** While technically mutable, instances are treated as immutable value objects after creation or deserialization.
- **Protocol-Managed:** Its lifecycle and serialization are managed entirely by the higher-level protocol framework, not by application-level code.
- **Efficiency:** The serialization format uses a bitmask (nullBits) to efficiently encode the presence of nullable fields, minimizing payload size for optional data.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by an Asset Editor service or controller in response to a user action, such as selecting a dataset to load from a UI. The `name` field is populated with the identifier of the requested asset.
    - **Server-Side:** Instantiated by the protocol's packet decoder when an incoming byte stream with Packet ID 333 is identified. The `deserialize` method is invoked to populate the instance from the network buffer.
- **Scope:** Extremely short-lived and transaction-scoped. An instance exists only for the brief period it takes to be serialized into a network buffer (client) or for its data to be read by a packet handler (server). It is immediately eligible for garbage collection after the network operation is complete.
- **Destruction:** Managed automatically by the Java garbage collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: the nullable String `name`. This state represents the entire payload of the request. The object contains no caching or derived state.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization. It is designed to be created, processed, and discarded within the confines of a single thread, typically a Netty I/O worker thread or a dedicated game logic thread.

**WARNING:** Sharing an instance of AssetEditorRequestDataset across multiple threads will lead to race conditions and unpredictable behavior. It must be confined to the thread that creates or deserializes it.

## API Surface
The primary contract is defined by the static utility methods for protocol interaction and the `Packet` interface methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorRequestDataset | O(N) | Constructs a new instance by reading from a ByteBuf. N is the length of the name string. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf. N is the length of the name string. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. N is the length of the name string. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a lightweight, non-allocating check on a buffer to verify if it contains a valid packet structure. Does not parse the full string. |

## Integration Patterns

### Standard Usage
The object is created and immediately passed to the network layer for dispatch. The application logic does not interact with serialization methods directly.

```java
// Client-side code requesting a dataset named "terrain.blocks"
AssetEditorRequestDataset request = new AssetEditorRequestDataset("terrain.blocks");

// The network service is responsible for serialization and sending
networkService.sendPacket(request);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not modify and re-send the same packet instance. This can cause issues in asynchronous network systems. Always create a new instance for each logical request.
- **Manual Serialization:** Application code should never call `serialize` or `deserialize`. These methods are strictly for use by the internal protocol engine.
- **Multi-threaded Access:** Do not access a packet instance from a different thread after it has been passed to the network layer.

## Data Pipeline
The AssetEditorRequestDataset is a transient data structure that flows through the network stack.

> **Flow (Client to Server):**
> User Action (UI) -> Asset Editor Service -> **new AssetEditorRequestDataset("name")** -> Protocol Encoder -> Serialized Byte Stream -> Network -> Server Protocol Decoder -> **Deserialized AssetEditorRequestDataset** -> Packet Handler -> Asset Loading Logic

