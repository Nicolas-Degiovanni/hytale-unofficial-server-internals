---
description: Architectural reference for PacketStatsRecorder
---

# PacketStatsRecorder

**Package:** com.hypixel.hytale.protocol.io
**Type:** Service Contract

## Definition
```java
// Signature
public interface PacketStatsRecorder {

   // A key to attach a recorder instance to a Netty Channel
   AttributeKey<PacketStatsRecorder> CHANNEL_KEY = AttributeKey.valueOf("PacketStatsRecorder");

   // A no-operation implementation to avoid null checks
   PacketStatsRecorder NOOP = new NoopPacketStatsRecorder();

   void recordSend(int packetId, int uncompressedSize, int compressedSize);

   void recordReceive(int packetId, int uncompressedSize, int compressedSize);

   @Nonnull
   PacketStatsRecorder.PacketStatsEntry getEntry(int packetId);

   // Nested data-access interfaces and records
   public interface PacketStatsEntry { ... }
   public record RecentStats(...) { ... }
}
```

## Architecture & Concepts
The PacketStatsRecorder is a contract for a diagnostic service that monitors network traffic at the packet level. It is not a standalone system but rather a component designed to be deeply embedded within the Netty networking pipeline. Its primary function is to provide a mechanism for collecting and querying metrics about individual packet types, such as frequency, size, and compression ratios.

Architecturally, this interface serves as a **Strategy Pattern**. It allows the engine to switch between different recording implementations. For example, a fully-featured recorder can be used in development or debug builds, while the provided NOOP (No-Operation) implementation can be used in production environments to eliminate any performance overhead associated with statistics collection.

The most critical architectural element is the static `CHANNEL_KEY`. This is a Netty `AttributeKey`, which signals that an instance of a PacketStatsRecorder is intended to be attached directly to a Netty `Channel`. This design decision scopes all statistics to a specific network connection, ensuring that data from different clients or server connections are not intermingled.

## Lifecycle & Ownership
- **Creation:** An implementation of PacketStatsRecorder is instantiated and attached to a Netty `Channel` during the connection initialization phase, typically within a `ChannelInitializer`. The specific implementation (e.g., a real recorder or the NOOP instance) is decided at this stage based on system configuration.
- **Scope:** The recorder's lifecycle is strictly bound to its associated Netty `Channel`. It persists for the entire duration of a single network session.
- **Destruction:** The object is eligible for garbage collection when the `Channel` is closed and its attribute map is cleared. There is no explicit `destroy` or `close` method on the interface; its memory is managed by the JVM.

## Internal State & Concurrency
- **State:** Any non-trivial implementation of this interface is inherently stateful and mutable. It must maintain internal data structures, such as a map or an array, to aggregate statistics for each packet ID over the lifetime of the connection.
- **Thread Safety:** **CRITICAL:** Implementations of this interface **must be thread-safe**. The `recordSend` and `recordReceive` methods will be called from a Netty I/O thread (EventLoop). Diagnostic tools or UI elements may call `getEntry` from a different thread (e.g., the main application thread). Failure to use concurrent collections or proper synchronization will lead to data corruption, race conditions, and potential deadlocks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| recordSend(id, uncomp, comp) | void | O(1) to O(log N) | Atomically updates statistics for an outbound packet. Must be non-blocking. |
| recordReceive(id, uncomp, comp) | void | O(1) to O(log N) | Atomically updates statistics for an inbound packet. Must be non-blocking. |
| getEntry(id) | PacketStatsEntry | O(1) to O(log N) | Retrieves a snapshot of the statistics for a given packet ID. |

## Integration Patterns

### Standard Usage
The recorder is not meant to be used by general game logic. Its primary consumers are low-level network handlers responsible for encoding and decoding packets. It is always retrieved from the channel's attribute map.

```java
// Example from within a Netty ChannelHandler
// Note: ctx is the ChannelHandlerContext

PacketStatsRecorder recorder = ctx.channel().attr(PacketStatsRecorder.CHANNEL_KEY).get();

// The NOOP implementation ensures recorder is never null if the key is unset,
// but defensive checking is still best practice.
if (recorder != null) {
    // This would be called by the protocol encoder after a packet is serialized
    recorder.recordSend(packet.getId(), uncompressedSize, compressedSize);
}
```

### Anti-Patterns (Do NOT do this)
- **Global Recorders:** Do not create a static or singleton recorder to be shared across all connections. Statistics are per-channel, and using a global instance would incorrectly merge data from all active sessions.
- **Assuming a Concrete Type:** Always program against the `PacketStatsRecorder` interface. Never cast it to a specific implementation. This preserves the ability to substitute the `NOOP` recorder.
- **Blocking Operations:** Implementations must not perform any blocking I/O or long-running computations. The recorder operates on the critical network I/O path, and any delay will directly increase network latency and reduce throughput.

## Data Pipeline
The PacketStatsRecorder acts as a passive "tap" on the data pipeline. It observes metadata about packets but does not modify the packets themselves.

**Outbound Flow:**
> Game Logic -> Packet Serialization -> Protocol Encoder -> **PacketStatsRecorder.recordSend** -> Netty Channel Write

**Inbound Flow:**
> Netty Channel Read -> Packet Deserialization -> Protocol Decoder -> **PacketStatsRecorder.recordReceive** -> Game Logic Handler

