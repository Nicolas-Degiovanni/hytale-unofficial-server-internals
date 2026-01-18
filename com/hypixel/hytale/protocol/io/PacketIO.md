---
description: Architectural reference for PacketIO
---

# PacketIO

**Package:** com.hypixel.hytale.protocol.io
**Type:** Utility

## Definition
```java
// Signature
public final class PacketIO {
```

## Architecture & Concepts

PacketIO is the foundational serialization and deserialization engine for the Hytale network protocol. It serves as the critical, low-level bridge between high-level Java Packet objects and the raw byte streams managed by the Netty network framework. This class is not a service or a component with its own state; it is a pure utility providing a static interface for protocol-specific data marshalling.

Its responsibilities are twofold:

1.  **Primitive & Complex Type I/O:** It provides a comprehensive suite of static methods for reading and writing protocol-defined data types from and to a Netty ByteBuf. This includes variable-length integers (via the VarInt class), fixed and variable-length strings, UUIDs, and specialized numeric types like half-precision floats.

2.  **Packet Framing & Lifecycle:** Its most critical function is to manage the entire lifecycle of a network packet frame. The methods writeFramedPacket and readFramedPacket orchestrate a multi-step process that includes packet ID lookup, payload serialization, optional Zstd compression, and length-prefixing. This ensures that all packets sent over the wire conform to a consistent and well-defined structure.

PacketIO is tightly coupled with the PacketRegistry, which it uses as a source of truth for packet metadata. Before serializing a packet, it queries the PacketRegistry to determine the packet's unique network ID, its maximum allowed size, and whether its payload should be compressed.

## Lifecycle & Ownership

-   **Creation:** This is a stateless utility class with a private constructor. It is **never instantiated**.
-   **Scope:** Its static methods are globally available and can be accessed throughout the application's lifetime.
-   **Destruction:** As a class consisting solely of static methods and constants, there is no instance-level state to manage. No destruction or cleanup logic is required.

## Internal State & Concurrency

-   **State:** PacketIO is entirely **stateless and immutable**. All methods are pure functions that operate exclusively on the arguments provided, primarily the ByteBuf buffer. It holds no internal caches or mutable fields.

-   **Thread Safety:** The class is inherently **thread-safe**. Its methods can be safely invoked from any thread without risk of race conditions within PacketIO itself.

    **WARNING:** While the class is thread-safe, the ByteBuf instances passed to its methods are not. The caller is responsible for ensuring that access to a given ByteBuf is properly synchronized. In a standard Netty application, this is guaranteed by the single-threaded event loop model. Do not share and modify a single ByteBuf across multiple threads without external locking.

## API Surface

The public API is divided into low-level type handlers and high-level frame handlers. The frame handlers are the primary integration point for the network layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| writeFramedPacket(packet, packetClass, out, stats) | void | O(N) | Orchestrates the full serialization, compression, and framing of a Packet object into the output ByteBuf. Throws ProtocolException on failure. |
| readFramedPacket(in, length, stats) | Packet | O(N) | Reads a complete packet frame from the input ByteBuf, performing decompression and deserialization. Throws ProtocolException on unknown ID or data corruption. |
| writeVarString(buf, value, maxLength) | void | O(L) | Writes a length-prefixed UTF-8 string. L is the length of the string. |
| readVarString(buf, offset) | String | O(L) | Reads a length-prefixed UTF-8 string from a given offset. |
| writeUUID(buf, value) | void | O(1) | Writes a 16-byte UUID to the buffer. |
| readUUID(buf, offset) | UUID | O(1) | Reads a 16-byte UUID from the buffer. |

*N = size of packet payload in bytes. L = length of string.*

## Integration Patterns

### Standard Usage

PacketIO should be used exclusively by components within the network pipeline, such as a Netty ChannelHandler responsible for encoding and decoding messages. The handler decodes the frame length, then passes the payload slice to PacketIO.

```java
// Example of an outbound message encoder (conceptual)
public class PacketEncoder extends MessageToByteEncoder<Packet> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Packet msg, ByteBuf out) {
        // The PacketStatsRecorder would be a shared service
        PacketStatsRecorder stats = getStatsRecorder();

        // Delegate the entire serialization and framing process to PacketIO
        PacketIO.writeFramedPacket(msg, msg.getClass(), out, stats);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Manual Framing:** Never attempt to manually write a packet's ID and length before serializing its payload. Always use writeFramedPacket to ensure the correct framing, compression, and statistical recording is performed. Manual implementation will inevitably lead to protocol desynchronization.

-   **Ignoring Buffer Ownership:** The ByteBuf instances passed to PacketIO are owned by the Netty framework. Do not hold references to them beyond the scope of the current network event. PacketIO methods that return a new ByteBuf (e.g., during decompression) transfer ownership to the caller, who is then responsible for its release.

-   **Incorrect String Methods:** Be vigilant about using the correct string I/O method. Using writeVarString for a field that the protocol defines as fixed-length will cause catastrophic deserialization failures.

## Data Pipeline

PacketIO is the central transformation component in the network data pipeline.

**Outbound Data Flow (Serialization):**

> Flow:
> Game Logic -> Packet Object -> **PacketIO.writeFramedPacket** -> (Serialization -> Zstd Compression) -> Framed ByteBuf -> Netty Channel

**Inbound Data Flow (Deserialization):**

> Flow:
> Netty Channel -> Framed ByteBuf -> **PacketIO.readFramedPacket** -> (Zstd Decompression -> Deserialization) -> Packet Object -> Game Logic

