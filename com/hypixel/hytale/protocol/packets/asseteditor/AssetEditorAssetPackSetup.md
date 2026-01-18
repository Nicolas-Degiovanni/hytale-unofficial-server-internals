---
description: Architectural reference for AssetEditorAssetPackSetup
---

# AssetEditorAssetPackSetup

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorAssetPackSetup implements Packet {
```

## Architecture & Concepts
The AssetEditorAssetPackSetup packet is a specialized Data Transfer Object (DTO) within the Hytale network protocol, identified by the static ID 314. Its sole purpose is to transmit the complete configuration of available asset packs from a server to a client when an asset editor session is initiated.

This packet acts as a manifest-of-manifests. It contains a map of asset pack identifiers to their corresponding AssetPackManifest objects. Upon receipt, the client's asset editing framework uses this data to bootstrap its environment. This includes populating UI elements with available assets, understanding versioning, and loading the necessary resources for editing.

This is a foundational, one-shot packet for session initialization. It is not used for incremental updates or real-time asset changes. Its transmission signals the start of an asset editing workflow, providing the client with the complete world-view of moddable and editable content.

### Lifecycle & Ownership
- **Creation:**
    - **Receiving End:** Instantiated by the protocol's deserialization layer when an incoming network buffer is identified with packet ID 314. The static `deserialize` method is the designated factory.
    - **Sending End:** Manually instantiated by server-side logic, typically within an asset management service, before being passed to the network layer for serialization.
- **Scope:** The object's lifetime is exceptionally brief and confined to the scope of a single network event processing cycle. It exists only to carry data from the network buffer to the application logic that consumes it.
- **Destruction:** The object is eligible for garbage collection immediately after the responsible packet handler or event listener has finished processing it. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is a mutable container for a nullable `Map<String, AssetPackManifest>`. Its state is defined entirely at the moment of creation, either through its constructor on the sending side or via the `deserialize` method on the receiving side. The presence of a deep `clone` method indicates that its state is complex and designed to be copied, not shared.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, processed, and discarded within a single thread, typically a Netty I/O worker thread.

    **Warning:** Sharing an instance of AssetEditorAssetPackSetup across multiple threads without explicit external synchronization is a severe anti-pattern and will lead to memory consistency errors and race conditions. Treat instances as thread-confined.

## API Surface
The public API is dominated by static methods for network serialization and validation, reflecting its role as a protocol-defined message.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorAssetPackSetup | O(N) | Constructs a new packet by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the internal state of the packet into a ByteBuf for network transmission. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the packet. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of a buffer to ensure it contains a structurally valid packet without full deserialization. |

*N = The total byte size of the serialized map data.*

## Integration Patterns

### Standard Usage
The primary integration pattern is with a network event loop and a corresponding handler. The packet is never managed directly by application code but is instead an artifact of the protocol layer.

**Receiving and Processing:**
```java
// In a network handler, after the packet has been deserialized
void handlePacket(AssetEditorAssetPackSetup setupPacket) {
    if (setupPacket.packs == null) {
        // Handle empty or cleared setup
        return;
    }
    
    AssetEditorService editorService = context.getService(AssetEditorService.class);
    editorService.initializeAssetPacks(setupPacket.packs);
}
```

**Sending a Configuration:**
```java
// On the server, preparing to send the configuration
Map<String, AssetPackManifest> manifests = assetRepository.getAllManifests();
AssetEditorAssetPackSetup setupPacket = new AssetEditorAssetPackSetup(manifests);

// The network layer will then call setupPacket.serialize(buffer)
networkChannel.send(setupPacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the `packs` map after the packet has been received. The object should be treated as immutable once it has been passed to a handler. If modifications are needed, create a defensive copy.
- **Instance Reuse:** Never reuse a packet instance to send multiple, distinct messages. A new instance must be created for each logical message to prevent state leakage and corruption.
- **Deserializing Untrusted Data:** Directly calling `deserialize` on a buffer from an untrusted source without prior length and content validation can expose the server to resource exhaustion vulnerabilities. The internal checks for dictionary and string sizes are critical security boundaries.

## Data Pipeline
The flow of this data object is linear and unidirectional, from the sender's application logic to the receiver's application logic, bridged by the network protocol.

> **Sending Flow:**
> Asset Editor Service -> `new AssetEditorAssetPackSetup(data)` -> **AssetEditorAssetPackSetup Instance** -> `serialize()` -> Network Buffer -> TCP/IP Stack

> **Receiving Flow:**
> TCP/IP Stack -> Network Buffer -> Packet Deserializer -> `deserialize()` -> **AssetEditorAssetPackSetup Instance** -> Packet Handler -> Asset Editor Service Update

