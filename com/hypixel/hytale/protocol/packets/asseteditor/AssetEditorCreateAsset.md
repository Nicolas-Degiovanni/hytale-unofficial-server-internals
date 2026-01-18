---
description: Architectural reference for AssetEditorCreateAsset
---

# AssetEditorCreateAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Structure

## Definition
```java
// Signature
public class AssetEditorCreateAsset implements Packet {
```

## Architecture & Concepts
The AssetEditorCreateAsset class is a network Data Transfer Object (DTO) that represents a command to create or overwrite a game asset on the server. It is a fundamental component of the real-time asset editing pipeline, enabling developers and content creators to push changes from a local tool directly into a running game instance without requiring a server restart.

This packet encapsulates all necessary information for the operation: the target asset's location (AssetPath), its raw binary content (data), and metadata for the transaction (token, rebuildCaches). Its design prioritizes network efficiency and server security through a custom binary serialization format. The format employs a fixed-size header containing a null-bit field and offsets to a variable-data section, minimizing bandwidth and allowing for fast, partial reads.

This class is not a service or manager; it is inert data. Its sole responsibility is to model the state of a single "create asset" request for transport over the network.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Client):** Instantiated directly via its constructor when an asset creation event is triggered (e.g., a user saves a file in an external editor). The caller is responsible for populating its public fields.
    - **Receiving Peer (Server):** Instantiated by the protocol layer's deserialization logic, specifically the static `deserialize` method. It is never created with `new` on the receiving side.

- **Scope:** Transient and short-lived. An instance exists only for the duration of a single network transaction. On the client, it is eligible for garbage collection after being passed to the network channel for serialization. On the server, it is eligible for garbage collection after its corresponding packet handler has finished processing the request.

- **Destruction:** Managed entirely by the Java Garbage Collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** Highly mutable. All primary data fields are public and can be modified directly after instantiation. The object holds the complete state for a single asset creation request, including potentially large byte arrays for the asset data itself. It performs no caching.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without external synchronization. It is designed to be created, populated, and processed by a single thread within the context of a network event loop (e.g., a Netty worker thread).

    **WARNING:** Modifying an AssetEditorCreateAsset instance after it has been submitted to the network layer for serialization will lead to race conditions and undefined behavior.

## API Surface
The public contract is defined by its fields, constructor, and the methods inherited from the Packet interface. The serialization and deserialization methods are the most critical components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorCreateAsset(...) | constructor | O(1) | Constructs a new packet instance. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into a custom binary format in the provided buffer. N is the size of the asset data. |
| deserialize(ByteBuf, int) | static AssetEditorCreateAsset | O(N) | Decodes binary data from the buffer into a new object instance. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a structural validation of the packet data without full deserialization. Checks offsets and lengths. |
| computeSize() | int | O(1) | Calculates the final serialized size of the packet in bytes. |

## Integration Patterns

### Standard Usage
This packet is constructed by a client-side system, sent over the network, and consumed by a server-side handler.

```java
// Client-side: Creating and sending the packet
AssetPath targetPath = new AssetPath("models", "item", "sword.json");
byte[] assetData = Files.readAllBytes(Paths.get("local/sword.json"));
int transactionToken = 12345;

AssetEditorCreateAsset packet = new AssetEditorCreateAsset(
    transactionToken,
    targetPath,
    assetData,
    new AssetEditorRebuildCaches(true, false),
    "ui.saveButton"
);

// The network client is responsible for calling packet.serialize()
// and writing the result to the channel.
networkClient.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not reuse packet instances for multiple requests. A new instance should be created for every asset creation command to ensure state integrity.
- **Ignoring Protocol Exceptions:** The `deserialize` method throws ProtocolException for malformed or malicious data. This exception **must** be caught at the network boundary to prevent server crashes and handle potential security threats, typically by disconnecting the offending client.
- **Manual Serialization:** Avoid calling `serialize` and managing the ByteBuf manually. Rely on the established network pipeline handlers which are designed to process Packet objects correctly.

## Data Pipeline
The flow of this data object is linear, moving from a client-side application logic layer to a server-side business logic layer via the network protocol stack.

> **Flow (Client to Server):**
> Asset Editor UI Event -> **AssetEditorCreateAsset (Instantiation)** -> Protocol Serializer -> Netty Channel -> Server Network Interface -> Netty Channel -> Protocol Deserializer -> **AssetEditorCreateAsset (Re-instantiation)** -> Asset Editor Packet Handler -> Filesystem Write & Cache Invalidation

