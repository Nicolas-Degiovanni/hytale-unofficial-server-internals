---
description: Architectural reference for AssetPath
---

# AssetPath

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AssetPath {
```

## Architecture & Concepts

The AssetPath class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol, specifically for systems interacting with the asset pipeline. It is not a service or manager, but rather a structured data container that represents a unique identifier for a game asset. An asset is uniquely identified by a combination of its source **pack** (e.g., the base game, a resource pack, or a mod) and its **path** within that pack.

The primary architectural significance of AssetPath lies in its highly optimized binary serialization format. To minimize network bandwidth, it does not use a generic serialization library. Instead, it implements a custom, byte-level layout:

1.  **Nullability Bitfield:** A single leading byte acts as a bitmask to indicate which of the variable-length fields (pack, path) are present in the payload. This avoids wasting bytes for null terminators or length prefixes for absent data.
2.  **Fixed-Size Header:** A fixed-size block follows the nullability field, containing integer offsets that point to the location of the variable-length data.
3.  **Variable-Data Block:** The actual string data for pack and path is written in a contiguous block at the end of the structure.

This design allows for extremely fast, zero-allocation validation and parsing directly from the network buffer, which is critical for a high-performance game server. It serves as a strict data contract between the client and server for any asset-related operations.

## Lifecycle & Ownership

-   **Creation:** AssetPath instances are created under two circumstances:
    1.  By the network protocol layer when `AssetPath.deserialize` is called to parse an incoming network packet.
    2.  By higher-level game logic when constructing an outbound packet that needs to reference a specific asset (e.g., `new AssetPath("hytale", "models/character.gltf")`).
-   **Scope:** This is a **transient** object. Its lifetime is strictly bound to the scope of a single network packet's processing or a single game logic operation. It is not designed to be cached or held for long periods.
-   **Destruction:** Instances are eligible for garbage collection immediately after the parent network packet has been processed or sent. There is no manual memory management or destruction logic.

## Internal State & Concurrency

-   **State:** The internal state consists of two public, nullable String fields: `pack` and `path`. The state is fully **mutable**. This design prioritizes performance and ease of use in single-threaded contexts over the safety of immutability.
-   **Thread Safety:** This class is **not thread-safe**. Direct access to its public fields from multiple threads without external synchronization will lead to race conditions. It is designed to be created, used, and discarded within the confines of a single thread, such as a Netty event loop thread or the main game thread. The static serialization and validation methods are safe to call from any thread, provided the ByteBuf they operate on is not being concurrently modified.

## API Surface

The public contract is focused entirely on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the given buffer according to the custom binary format. N is the combined length of the strings. |
| deserialize(ByteBuf, int) | AssetPath | O(N) | Decodes an AssetPath from the given buffer at a specific offset. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a series of non-deserializing checks on a buffer to verify a valid AssetPath structure. Critical for security and preventing parsing errors. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(ByteBuf, int) | int | O(1) | Reads the header from an existing buffer to determine the total size of a serialized AssetPath without parsing its contents. |

## Integration Patterns

### Standard Usage

AssetPath is almost never used in isolation. It is embedded within larger packet objects and handled by the protocol layer.

```java
// Example: Constructing a packet to request an asset from the server
RequestAssetPacket request = new RequestAssetPacket();
request.asset = new AssetPath("hytale.common", "textures/ui/button.png");

// The network layer would then serialize the entire packet
// which in turn calls request.asset.serialize(buffer);
networkManager.send(request);
```

### Anti-Patterns (Do NOT do this)

-   **Sharing Instances:** Do not share an AssetPath instance between threads. Its mutable nature makes it unsafe for concurrent access. If data must be passed to another thread, create a new copy.
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from an untrusted source without first calling `validateStructure`. Bypassing validation can result in uncaught ProtocolExceptions or other buffer-related errors that can crash a network thread.
-   **Manual Field Manipulation During I/O:** Do not modify the `pack` or `path` fields while the object is being serialized or after it has been prepared for sending. The computed size and offsets will become invalid.

## Data Pipeline

The primary role of AssetPath is to be a data structure that fluidly crosses the boundary between raw network bytes and structured game objects. The following illustrates the inbound data flow.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> **AssetPath.validateStructure** -> **AssetPath.deserialize** -> Parent Packet Object -> Game Event Bus -> Asset System Logic

