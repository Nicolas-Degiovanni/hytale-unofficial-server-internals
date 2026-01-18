---
description: Architectural reference for the AssetPart network packet.
---

# AssetPart

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetPart implements Packet {
```

## Architecture & Concepts
The AssetPart class is a fundamental Data Transfer Object within the Hytale network protocol layer. It is not a service or manager, but rather a structured container for a raw segment of a larger game asset. Its primary role is to facilitate the streaming of game resources, such as textures, models, or sounds, from the server to the client during the initial connection setup phase.

This packet is a critical component of the asset pre-warming and loading pipeline. By breaking large assets into smaller, manageable chunks encapsulated by AssetPart, the engine can stream resources over the network without blocking the connection or requiring a single, massive download. The protocol defines a maximum size of approximately 4MB per part, as indicated by the MAX_SIZE constant, which balances network efficiency with memory pressure.

The class implements the Packet interface, signifying its role as a serializable unit of data that can be processed by the Netty-based network pipeline. Its implementation includes low-level, performance-critical logic for serialization and deserialization directly against a Netty ByteBuf, using VarInts for length prefixing to conserve bandwidth.

## Lifecycle & Ownership
- **Creation:** An AssetPart instance is created under two distinct circumstances:
    1.  **Server-Side (Serialization):** Instantiated by an asset streaming service when a large game asset is chunked for network transmission. A new AssetPart is created to wrap each byte array segment.
    2.  **Client-Side (Deserialization):** Instantiated by the protocol's packet decoder when an incoming network buffer with Packet ID 25 is identified. The static `deserialize` method is invoked, which constructs the object from the raw bytes.

- **Scope:** The object's lifetime is exceptionally short and confined to the network processing pipeline. It is a transient object, designed to be created, processed, and then immediately discarded.

- **Destruction:** The object is managed by the Java Garbage Collector. Once the byte array payload has been extracted and passed to a higher-level asset reassembly system, the AssetPart instance becomes unreachable and is reclaimed. There are no manual cleanup or `close` operations required.

## Internal State & Concurrency
- **State:** The internal state consists of a single, mutable `byte[]` field named `part`. While technically mutable, this object should be treated as immutable after its creation, especially after deserialization on the client. The data it holds represents a point-in-time snapshot of a piece of an asset.

- **Thread Safety:** This class is **not thread-safe**. The public `part` field can be accessed and modified without synchronization. This is by design, as network packet processing in Netty is typically handled by a single event loop thread.

    **Warning:** If an AssetPart object is passed from the Netty I/O thread to a separate worker thread (e.g., for asset decompression or processing), the handoff must be done carefully. Either the raw byte array should be copied, or the receiving system must guarantee that no other thread will access the object. Modifying the internal byte array from multiple threads will lead to data corruption.

## API Surface
The public contract is dominated by static methods for protocol handling and instance methods for serialization, conforming to the Packet interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetPart | O(N) | Constructs an AssetPart from a ByteBuf. N is the size of the byte array. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. N is the size of the byte array. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this packet will occupy on the wire. Does not perform serialization. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight, read-only check to validate if a buffer contains a structurally valid AssetPart. Does not fully deserialize. |
| getId() | int | O(1) | Returns the static packet identifier, which is 25. |

## Integration Patterns

### Standard Usage
The class is intended to be used exclusively by the network protocol codec. A higher-level system, such as an AssetStreamingManager, receives the deserialized object from the network pipeline and processes its payload.

```java
// Executed within a Netty ChannelInboundHandler on the client
// The 'packet' object is provided by the upstream decoder.

if (packet instanceof AssetPart assetPart) {
    // The asset reassembly service is responsible for collecting
    // all parts and reconstructing the original asset.
    byte[] dataChunk = assetPart.part;
    if (dataChunk != null) {
        assetReassemblyService.processChunk(dataChunk);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Do not use `new AssetPart()` on the client side for any logic. These objects are meant to originate from the server and be created only through deserialization.
- **Long-Term Storage:** Do not hold references to AssetPart objects in caches or long-lived collections. Their payload can be large (~4MB), and retaining them will cause significant memory leaks. Extract the byte array and discard the packet object.
- **Payload Modification:** Do not modify the `part` byte array after it has been received. This data is a faithful segment of a server-side file and should be considered read-only.

## Data Pipeline
The AssetPart packet is a vehicle for data flowing from the server's asset system to the client's.

> **Flow:**
> Server Asset File -> File Chunker -> **new AssetPart(chunk)** -> Protocol Serializer -> TCP Stream -> Client Network Decoder -> **AssetPart.deserialize(buffer)** -> Asset Reassembly Service -> Final Asset in Memory

