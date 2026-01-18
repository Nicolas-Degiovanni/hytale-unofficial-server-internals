---
description: Architectural reference for AssetEditorCreateAssetPack
---

# AssetEditorCreateAssetPack

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorCreateAssetPack implements Packet {
```

## Architecture & Concepts
The AssetEditorCreateAssetPack class is a Data Transfer Object (DTO) that represents a specific command within the Hytale network protocol. It is not a service or manager; its sole purpose is to encapsulate the data required to request the creation of a new asset pack on a server. This packet is typically sent by a client, such as the official Hytale asset editor, to a game or asset server.

As an implementation of the Packet interface, this class adheres to a strict contract for network communication. It provides methods for serialization to a Netty ByteBuf, deserialization from a ByteBuf, and self-contained size calculation. The presence of static metadata fields like PACKET_ID and FIXED_BLOCK_SIZE is a common performance optimization in the Hytale protocol layer, allowing the network engine to process packets without relying on slower mechanisms like reflection.

The structure is designed for efficiency, using a bitmask (nullBits) to indicate the presence of optional, variable-sized fields like the manifest. This minimizes payload size when optional data is not included.

### Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **Client-Side (Sending):** Instantiated by application logic when a user action triggers the need to create an asset pack. The constructor is called, fields are populated, and the object is passed to the network layer for transmission.
    2.  **Server-Side (Receiving):** Instantiated by the protocol's deserialization engine when an incoming byte stream with PACKET_ID 316 is identified. The static `deserialize` method acts as the factory in this context.

- **Scope:** The object's lifetime is extremely short and bound to a single network operation. On the client, it exists only long enough to be serialized. On the server, it exists only for the duration of its processing by a corresponding packet handler.

- **Destruction:** The object is managed by the Java Garbage Collector. Once it falls out of scope (e.g., after the packet handler completes), it becomes eligible for collection. There is no manual resource management.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The public fields `token` and `manifest` can be modified after instantiation. This is a deliberate design choice for a DTO that is constructed and populated before being passed to another system. The state consists of an integer token and a nullable reference to an AssetPackManifest object.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, populated, and processed within a single thread, which is the standard model for most network packet processing pipelines (e.g., a Netty event loop).

    **Warning:** Sharing an AssetEditorCreateAssetPack instance across multiple threads is a critical anti-pattern. Modifying its state from one thread while another thread is serializing or reading it will result in data corruption and undefined behavior.

## API Surface
The public API is primarily concerned with object construction and the network protocol contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorCreateAssetPack() | constructor | O(1) | Creates an empty packet. |
| AssetEditorCreateAssetPack(token, manifest) | constructor | O(1) | Creates a fully populated packet. |
| deserialize(buf, offset) | static AssetEditorCreateAssetPack | O(N) | Factory method to construct a packet from a raw ByteBuf. N is the size of the manifest. |
| serialize(buf) | void | O(N) | Writes the packet's state into the provided ByteBuf. N is the size of the manifest. |
| computeSize() | int | O(N) | Calculates the total byte size the packet will occupy when serialized. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check to ensure the data in the buffer is a valid representation of this packet. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer and corresponding business logic handlers. A developer would not typically interact with this class unless implementing a custom client or server that speaks the Hytale protocol.

**Server-Side Packet Handler Example:**
```java
// Within a packet handler that receives a Packet object
if (packet instanceof AssetEditorCreateAssetPack createPacket) {
    int userToken = createPacket.token;
    AssetPackManifest manifest = createPacket.manifest;

    // Authenticate user with token and process the manifest
    assetService.createNewAssetPack(userToken, manifest);
}
```

**Client-Side Sending Example:**
```java
// Application logic prepares the data
AssetPackManifest myManifest = buildManifest();
int sessionToken = session.getToken();

// Packet is created and sent via a network service
AssetEditorCreateAssetPack packet = new AssetEditorCreateAssetPack(sessionToken, myManifest);
networkClient.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold onto and re-use packet instances for multiple messages. They are lightweight objects and should be created new for each distinct message to avoid state corruption.
- **Manual Serialization:** Avoid calling `serialize` directly. The network engine is responsible for managing the ByteBuf lifecycle. Passing a packet to the network layer is the correct abstraction.
- **Ignoring Nullability:** The `manifest` field is nullable. Always perform a null check before attempting to access it, as a packet may be validly sent without a manifest.

## Data Pipeline
The AssetEditorCreateAssetPack serves as a data container that moves a user's intent from the client application into the server's business logic.

> **Flow:**
> Client-Side User Action -> Application Logic creates **AssetEditorCreateAssetPack** -> Network Layer serializes packet -> Raw Bytes over TCP -> Server-Side Network Layer deserializes into **AssetEditorCreateAssetPack** -> Packet Handler -> Asset Service Logic

