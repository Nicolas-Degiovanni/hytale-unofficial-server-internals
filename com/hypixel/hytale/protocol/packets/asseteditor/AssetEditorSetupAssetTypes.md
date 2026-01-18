---
description: Architectural reference for AssetEditorSetupAssetTypes
---

# AssetEditorSetupAssetTypes

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorSetupAssetTypes implements Packet {
```

## Architecture & Concepts
The AssetEditorSetupAssetTypes class is a network packet data structure, not a service or manager. It serves as a Data Transfer Object (DTO) within the Hytale network protocol, specifically for initializing the in-game asset editor. Its sole responsibility is to encapsulate the list of available asset types that a client can edit, serializing this data for network transmission and deserializing it upon receipt.

As an implementation of the Packet interface, it adheres to a strict contract for network communication. This contract includes methods for serialization, deserialization, size computation, and structural validation. The class itself is stateless in terms of behavior; its value is entirely defined by the data it carries in the *assetTypes* field.

The static constants defined within the class, such as PACKET_ID and MAX_SIZE, provide critical metadata to the underlying protocol engine. This metadata enables the network layer to correctly route incoming byte streams to the appropriate deserializer and to perform initial sanity checks against malformed or malicious data payloads.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** An instance is created and populated by a higher-level service responsible for managing asset editor sessions. This object is then passed to the network pipeline for serialization.
    - **Receiving Peer (Client):** An instance is created by the protocol's deserialization layer. A packet dispatcher reads the PACKET_ID from the raw network buffer and invokes the static *deserialize* factory method on this class to construct the object from the byte stream.

- **Scope:** This object is ephemeral and has an extremely short lifecycle. It exists only for the duration of a single network transaction. Once its data has been consumed by the application logic (e.g., to populate a UI), it is no longer referenced and becomes eligible for garbage collection.

- **Destruction:** Cleanup is managed automatically by the Java garbage collector. There are no native resources or explicit destruction methods.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a single nullable array, *assetTypes*. The object's state is fully defined upon creation, either through its constructor or the *deserialize* method.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data container and provides no internal synchronization. It is designed to be created, processed, and discarded within a single-threaded context, such as a Netty I/O event loop.

    **WARNING:** Sharing an instance of this class across threads without external synchronization will lead to unpredictable behavior and data corruption. Do not modify its state after it has been passed to the network layer for transmission.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorSetupAssetTypes | O(N) | Constructs a new instance by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of the data in a ByteBuf to ensure it represents a valid packet structure without full deserialization. |
| clone() | AssetEditorSetupAssetTypes | O(N) | Creates a deep copy of the object and its contained asset types. |

*N = number of elements in the assetTypes array.*

## Integration Patterns

### Standard Usage
This packet is handled by the core protocol engine. Application-level code typically interacts with it inside a network event handler.

```java
// Example of a receiving-side handler
public void handlePacket(AssetEditorSetupAssetTypes packet) {
    if (packet.assetTypes == null) {
        // Handle the case where no asset types were sent
        return;
    }

    // The UI or asset management system consumes the data
    AssetEditorUI editor = getAssetEditorUI();
    editor.configureAssetTypes(packet.assetTypes);
}

// Example of sending-side logic
AssetEditorAssetType[] types = assetService.getAvailableTypes();
AssetEditorSetupAssetTypes packet = new AssetEditorSetupAssetTypes(types);
networkChannel.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Queuing:** Do not modify the *assetTypes* array after the packet instance has been passed to the network channel for sending. Serialization may occur on a different thread, leading to a race condition where a partially modified or inconsistent object state is sent over the network.

- **Instance Re-use:** Do not cache and re-use packet instances. They are lightweight objects intended for single-use. Create a new instance for each message to ensure state integrity.

- **Ignoring Validation Results:** On the receiving end, failing to use *validateStructure* on data from untrusted sources before attempting deserialization can expose the application to resource exhaustion attacks (e.g., by declaring an impossibly large array size) or other denial-of-service vulnerabilities.

## Data Pipeline
The primary role of this class is to act as a container for data moving between the server's game logic and the client's user interface.

> **Server-Side Flow (Sending):**
> Asset Management Service -> `new AssetEditorSetupAssetTypes(data)` -> Protocol Encoder -> **serialize()** -> Netty ByteBuf -> Network

> **Client-Side Flow (Receiving):**
> Network -> Netty ByteBuf -> Protocol Decoder -> **deserialize()** -> **AssetEditorSetupAssetTypes Instance** -> Application Event Bus -> Asset Editor UI Update

