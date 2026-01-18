---
description: Architectural reference for MapImage
---

# MapImage

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class MapImage {
```

## Architecture & Concepts
The MapImage class is a low-level Data Transfer Object designed for the efficient network transmission of world map tile data. It is not a general-purpose image container; its structure is tightly coupled to the Hytale network protocol and optimized for performance and minimal bandwidth usage.

This class acts as a passive data structure that represents the serialized form of a map segment. Its primary role is to be a concrete representation of data read from or written to a Netty ByteBuf. The serialization format is custom, employing a leading bitmask byte to denote the presence of nullable fields (in this case, the main data array) and variable-length integers (VarInt) to encode array lengths. This design minimizes payload size for sparse or empty map tiles.

All serialization and deserialization logic is self-contained within the class via static factory methods and instance methods, abstracting the raw byte manipulation from higher-level network handlers.

## Lifecycle & Ownership
- **Creation:** A MapImage instance is created under two circumstances:
    1.  **On Deserialization:** The primary creation path is via the static `deserialize` method, which is invoked by a network protocol decoder (e.g., a Netty ChannelInboundHandler) when a corresponding packet is received from the network.
    2.  **On Serialization:** An instance is created using its constructor on the sending side (typically the server) to encapsulate map data before it is passed to the `serialize` method for encoding.

- **Scope:** Transient. The lifetime of a MapImage object is extremely short. It is designed to exist only for the duration of processing a single network packet. Once its data has been consumed (e.g., loaded into a texture on the client), the object should be considered stale and is eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual resource management or cleanup methods. Holding long-term references to MapImage objects is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data container with public fields for width, height, and the pixel data array. Its state is intended to be populated once upon creation and then read by a consumer.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms. All methods, including `serialize` and `deserialize`, assume they are operating on a ByteBuf within a single thread, typically a Netty I/O worker thread. Concurrent access to a MapImage instance or its underlying data array from multiple threads will result in undefined behavior and data corruption. Any multi-threaded processing requires external synchronization managed by the caller.

## API Surface
The public API is designed for interaction with the network protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static MapImage | O(N) | Constructs a MapImage object by reading from a ByteBuf. N is the number of pixels. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided ByteBuf. N is the number of pixels. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(1) | Calculates the number of bytes this object will consume when serialized. Essential for pre-allocating buffers. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a non-allocating check to verify if a buffer contains a structurally valid MapImage. Used for fast-path rejection of invalid packets. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the exact size of a serialized MapImage within a buffer without full deserialization. Used by decoders to advance the buffer read index. |

## Integration Patterns

### Standard Usage
The class is intended to be used by network handlers to decode incoming data or prepare outgoing data. The lifecycle is typically contained within a single method.

```java
// Example within a Netty ChannelInboundHandler
void channelRead(ChannelHandlerContext ctx, ByteBuf in) {
    // 1. Validate the buffer contains enough data for a valid object
    if (MapImage.validateStructure(in, in.readerIndex()).isError()) {
        // Handle error, close connection, etc.
        return;
    }

    // 2. Deserialize the object from the buffer
    MapImage mapTile = MapImage.deserialize(in, in.readerIndex());

    // 3. Advance the buffer's reader index
    in.readerIndex(in.readerIndex() + mapTile.computeSize());

    // 4. Pass the data to the game engine for processing
    gameEngine.getWorldMap().updateTile(mapTile);

    // The mapTile object now goes out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not retain and modify a MapImage instance over time or across multiple packets. It is a transient DTO, not a long-lived stateful object.

- **Multi-threaded Access:** Do not share a MapImage instance between threads. For example, do not deserialize on a network thread and hand the object directly to a separate rendering thread without a deep copy or proper synchronization.

- **Ignoring Validation:** Bypassing `validateStructure` before calling `deserialize` on untrusted data exposes the application to potential ProtocolExceptions, such as out-of-memory errors from maliciously crafted packets declaring a massive array size.

## Data Pipeline
MapImage serves as a critical link in the data flow between the server's world state and the client's visual representation.

> **Server-Side Flow:**
> World Generation/State Change -> **MapImage (new)** -> **MapImage.serialize()** -> Network Packet Encoder -> TCP/IP Stack

> **Client-Side Flow:**
> TCP/IP Stack -> Network Packet Decoder -> **MapImage.deserialize()** -> World Map System -> Texture Atlas Update -> GPU Render

