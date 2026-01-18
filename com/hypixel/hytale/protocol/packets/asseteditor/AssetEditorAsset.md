---
description: Architectural reference for AssetEditorAsset
---

# AssetEditorAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Packet Data Structure

## Definition
```java
// Signature
public class AssetEditorAsset {
```

## Architecture & Concepts

The AssetEditorAsset class is a fundamental data structure, not a service or manager. It serves as a Data Transfer Object (DTO) specifically designed for the Hytale network protocol. Its primary role is to define the in-memory representation and the corresponding on-the-wire binary format for a piece of asset metadata, namely its content hash and its file path.

This class is a component within the protocol's serialization and deserialization pipeline. It does not operate independently but is typically embedded within larger, more comprehensive packet objects.

The core architectural pattern is a high-performance, custom binary serialization format built on Netty's ByteBuf. The binary layout is optimized for network efficiency and consists of three main parts:
1.  **Nullable Bit Field:** A single byte at the start of the structure acts as a bitmask to indicate which of the subsequent fields are present (not null). This avoids wasting bytes for null data.
2.  **Fixed-Size Block:** A block containing fixed-size data. In this case, it holds 4-byte integer offsets pointing to the location of variable-length data.
3.  **Variable-Size Block:** A contiguous block of memory at the end of the structure where all variable-length data, such as strings and nested objects, are written sequentially.

This design allows for extremely fast parsing and validation, as the structure of the entire object can be understood by first reading the small, fixed-size header.

## Lifecycle & Ownership

-   **Creation:** Instances are created under two primary circumstances:
    1.  **Deserialization:** The static deserialize method is invoked by a parent packet's deserialization logic when an incoming network message is being parsed from a ByteBuf.
    2.  **Manual Instantiation:** Game logic creates a new instance when preparing an outbound packet that needs to communicate asset information.
-   **Scope:** **Transient**. The lifetime of an AssetEditorAsset object is short and is strictly bound to the lifecycle of the network packet it belongs to. It is created, processed, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this class.

## Internal State & Concurrency

-   **State:** The class is a **mutable** data container. Its public fields, hash and path, can be directly accessed and modified after instantiation. It holds no internal caches or derived state; it is a direct representation of its data.

-   **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access or modification from multiple threads. It is designed to be created, populated, and read within a single-threaded context, such as a Netty I/O worker thread or the main game logic thread.

    **WARNING:** Sharing an instance of AssetEditorAsset across threads without external synchronization will lead to race conditions and unpredictable behavior.

## API Surface

The public API is focused on serialization, deserialization, and structural validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorAsset | O(N) | **Primary Deserializer.** Constructs an instance from a binary representation within a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | **Primary Serializer.** Writes the object's state into the provided ByteBuf according to the custom binary protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will consume when serialized. Crucial for pre-allocating buffers. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a structural integrity check on the binary data in a buffer without performing a full deserialization. Used for fast validation of untrusted data. |
| clone() | AssetEditorAsset | O(1) | Creates a shallow copy of the object. Note that the contained AssetPath object is cloned, but the hash String is shared. |

## Integration Patterns

### Standard Usage

AssetEditorAsset is not used as a standalone object. It is constructed and embedded within a parent packet before being sent over the network, or it is received as the result of deserializing a parent packet.

```java
// Example: Creating an asset reference to send
AssetPath path = new AssetPath("models/item/sword.json");
String contentHash = "d8e8fca2dc0f896fd7cb4cb0031ba249";

AssetEditorAsset assetToSend = new AssetEditorAsset(contentHash, path);

// This object would then be added to a larger packet, for example:
// OutboundPacket.addAsset(assetToSend);
// networkManager.send(OutboundPacket);

// The serialization is handled by the parent packet's logic
ByteBuf buffer = Unpooled.buffer();
assetToSend.serialize(buffer);
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term State:** Do not use this class to store application state. Its purpose is for transient data transfer only. Storing it in a long-lived collection is a misuse of its design.
-   **Concurrent Modification:** Do not pass an instance to another thread for modification without explicit locking. The object is not designed for concurrent access.
-   **Ignoring Validation:** When processing incoming data from an untrusted source, always use validateStructure before attempting to deserialize. A malicious payload could otherwise trigger exceptions or excessive memory allocation.

## Data Pipeline

The AssetEditorAsset class is a node in the network data serialization and deserialization pipeline.

> **Outbound Flow (Serialization):**
> Game Logic -> `new AssetEditorAsset()` -> Parent Packet -> **AssetEditorAsset.serialize()** -> Netty ByteBuf -> Network Socket

> **Inbound Flow (Deserialization):**
> Network Socket -> Netty ByteBuf -> Parent Packet Parser -> **AssetEditorAsset.deserialize()** -> Game Logic

