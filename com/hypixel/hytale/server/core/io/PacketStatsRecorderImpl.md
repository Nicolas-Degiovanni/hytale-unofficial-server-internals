---
description: Architectural reference for PacketStatsRecorderImpl
---

# PacketStatsRecorderImpl

**Package:** com.hypixel.hytale.server.core.io
**Type:** Transient

## Definition
```java
// Signature
public class PacketStatsRecorderImpl implements PacketStatsRecorder {
```

## Architecture & Concepts
The PacketStatsRecorderImpl is a high-performance, thread-safe data collector responsible for tracking detailed statistics about network packets sent and received by the server. It serves as a critical observability tool for network performance analysis, debugging, and server health monitoring.

Architecturally, this class is designed for minimal overhead. It acts as a side-channel sink for metrics, invoked directly from the core network I/O pipeline. Its primary design feature is a pre-allocated, fixed-size array of PacketStatsEntry objects, indexed directly by the packet ID. This approach avoids dynamic memory allocation and locking during the critical path of recording packet data, ensuring that performance monitoring does not degrade network throughput.

Each PacketStatsEntry is an independent, concurrent data structure that aggregates metrics for a single packet type. This includes counts, total bytes (compressed and uncompressed), min/max sizes, and rolling averages.

A key integration point is the static METRICS_REGISTRY. This exposes the collected statistics to Hytale's central metrics system, allowing the data to be queried, serialized, and potentially shipped to external monitoring dashboards without direct coupling to the recorder instance itself.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor. It is typically created once by a higher-level manager that oversees a network session or the entire server instance, such as a connection manager or the main server bootstrap process.
- **Scope:** The instance persists for the lifetime of the component it is monitoring. For a server-wide recorder, this would be the entire server session.
- **Destruction:** The object is eligible for garbage collection when its owner is destroyed. It holds no native resources and does not require an explicit destruction or close method.

## Internal State & Concurrency
- **State:** This class is fundamentally stateful and mutable. Its core state is the `entries` array, which holds 512 PacketStatsEntry objects. Each entry, in turn, maintains a rich set of mutable atomic counters and concurrent collections for its respective packet type.

- **Thread Safety:** This class is designed to be fully thread-safe and is expected to be called from multiple network I/O threads simultaneously.
    - **Write Operations:** The `recordSend` and `recordReceive` methods are lock-free. They delegate to the appropriate PacketStatsEntry based on the packet ID. Since each thread writing data for a specific packet ID interacts with the same entry, the internal concurrency of PacketStatsEntry is critical.
    - **Internal Concurrency:** The PacketStatsEntry class achieves thread safety through the exclusive use of atomic primitives (AtomicInteger, AtomicLong) and concurrent data structures (ConcurrentLinkedQueue). This ensures that all increments, aggregations, and updates are atomic operations, preventing race conditions and data corruption without the performance penalty of explicit locks.
    - **Read Operations:** All getter methods on PacketStatsEntry rely on atomic reads, providing safe, up-to-date values. Iteration over the `sentRecently` and `receivedRecently` queues for calculating recent stats is also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| recordSend(int, int, int) | void | O(1) | Records metrics for an outgoing packet. This is a fire-and-forget, lock-free operation. |
| recordReceive(int, int, int) | void | O(1) | Records metrics for an incoming packet. This is a fire-and-forget, lock-free operation. |
| getEntry(int) | PacketStatsEntry | O(1) | Retrieves the dedicated statistics entry for a given packet ID. |

## Integration Patterns

### Standard Usage
This component is intended to be injected into the network packet processing pipeline. After a packet is encoded (for sending) or decoded (for receiving), the pipeline should immediately call the appropriate record method.

```java
// Within a network pipeline handler...
PacketStatsRecorderImpl recorder = networkSession.getStatsRecorder();

// After encoding a packet for sending
int packetId = packet.getId();
int uncompressedSize = ...;
int compressedSize = ...;
recorder.recordSend(packetId, uncompressedSize, compressedSize);
```

### Anti-Patterns (Do NOT do this)
- **Multiple Instances:** Do not create more than one PacketStatsRecorderImpl for a single monitored entity (e.g., a server). Doing so will result in fragmented and incomplete statistics. A single instance should be created and shared.
- **Unbounded Packet IDs:** This implementation assumes packet IDs are within the range of 0-511. Sending a packet ID outside this range will result in the record being silently ignored. The calling code must validate packet IDs.
- **External Modification:** Do not attempt to modify the state of a PacketStatsEntry externally, for example by calling its internal `reset` method without proper synchronization with the network threads. This can lead to data loss.

## Data Pipeline
PacketStatsRecorderImpl does not participate in the primary data flow of packets. Instead, it acts as a terminal sink for metadata *about* the packets.

> **Write Flow:**
> Network I/O Thread -> Packet Encoder -> **PacketStatsRecorderImpl.recordSend()**

> **Read Flow:**
> Metrics System -> **PacketStatsRecorderImpl.METRICS_REGISTRY** -> Query & Serialize Stats -> Monitoring Endpoint

