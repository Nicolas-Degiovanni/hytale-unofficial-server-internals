---
description: Architectural reference for AssetEditorRequestChildrenList
---

# AssetEditorRequestChildrenList

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AssetEditorRequestChildrenList implements Packet {
```

## Architecture & Concepts
The AssetEditorRequestChildrenList class is a network packet, a fundamental Data Transfer Object (DTO) within the Hytale protocol. It represents a client-to-server request for a directory listing within the game's asset database. This packet is a key component of the remote asset editing and browsing system, likely used by tools such as the Hytale Model Maker.

Its primary role is to transport a single piece of information: the `AssetPath` of a directory whose children the client wishes to enumerate. The server-side packet handler receives this request, queries the underlying asset storage system for the specified path, and sends a corresponding response packet (e.g., AssetEditorChildrenListResponse) containing the list of child assets.

As an implementation of the Packet interface, this class adheres to a strict contract for network serialization and deserialization, ensuring binary compatibility between the client and server. The serialization logic is notable for its use of a bit field (`nullBits`) to efficiently encode the presence or absence of its nullable `path` field, a common optimization pattern in the Hytale protocol to minimize payload size. A null path likely signifies a request for the root-level asset directories.

## Lifecycle & Ownership
-   **Creation:** An instance is created on the client (e.g., an external editing tool) in response to a user action, such as expanding a folder in an asset browser UI. The client-side logic constructs the packet, populating the `path` field with the target directory.
-   **Scope:** This object has an extremely brief, ephemeral lifecycle. On the client, it exists only long enough to be serialized into a Netty ByteBuf by the network pipeline. On the server, it is deserialized from a ByteBuf, processed by a single handler, and then becomes eligible for garbage collection. It is not designed to be stored or referenced long-term.
-   **Destruction:** The object is managed by the Java Garbage Collector. It is dereferenced and destroyed immediately after its data has been read (server-side) or written to the network buffer (client-side).

## Internal State & Concurrency
-   **State:** The class holds a single mutable field, `path`, which is an instance of AssetPath. The state is simple and is intended to be set once at construction. The object itself is a plain data container with no internal logic beyond serialization.
-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms and is designed for use within a single-threaded context, such as a Netty event loop. It must not be shared or modified across multiple threads. This is a standard and acceptable design for network DTOs, which are processed sequentially.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (321). |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided network buffer. |
| deserialize(ByteBuf, int) | AssetEditorRequestChildrenList | O(N) | Static factory method. Decodes a new instance from a network buffer at a given offset. |
| computeSize() | int | O(N) | Calculates the number of bytes this packet will occupy when serialized. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Statically validates if the data in a buffer represents a well-formed packet of this type. |

*Complexity O(N) refers to the depth and component count of the contained AssetPath.*

## Integration Patterns

### Standard Usage
This packet is created and dispatched by a higher-level service responsible for communicating with the server's asset editor endpoint. Direct manipulation is rare; developers interact with a service that abstracts packet creation and network transmission.

```java
// Client-side service logic
// Assume 'assetEditorService' handles network communication

AssetPath targetDirectory = new AssetPath("models/monsters/creeper");
AssetEditorRequestChildrenList request = new AssetEditorRequestChildrenList(targetDirectory);

// The service would serialize and send this packet to the server
assetEditorService.sendPacket(request);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the `path` field after the packet has been passed to a network service for sending. The object should be treated as immutable post-construction to prevent race conditions where the data is changed during asynchronous serialization.
-   **Instance Reuse:** Do not cache and reuse packet instances for multiple requests. Each request to the server must be represented by a new, distinct AssetEditorRequestChildrenList object.
-   **Manual Serialization:** Application-level code should never call `serialize` or `deserialize` directly. These methods are strictly for use by the underlying network protocol codecs in the Netty pipeline.

## Data Pipeline
The flow of data for an asset listing request is a standard client-server request/response cycle mediated by this packet.

> **Flow:**
> Client UI Event (Folder expanded) -> Asset Editor Service -> **new AssetEditorRequestChildrenList(path)** -> Protocol Encoder -> Network -> Protocol Decoder -> **AssetEditorRequestChildrenList instance** -> Server Packet Handler -> Asset Database Query -> Response Packet Sent

