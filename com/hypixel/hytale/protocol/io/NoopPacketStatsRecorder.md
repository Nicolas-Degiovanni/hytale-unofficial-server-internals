---
description: Architectural reference for NoopPacketStatsRecorder
---

# NoopPacketStatsRecorder

**Package:** com.hypixel.hytale.protocol.io
**Type:** Utility

## Definition
```java
// Signature
final class NoopPacketStatsRecorder implements PacketStatsRecorder {
```

## Architecture & Concepts
The NoopPacketStatsRecorder is a concrete implementation of the PacketStatsRecorder interface that performs no operations. It embodies the **Null Object Pattern**, providing a functional, non-null default that can be used when network statistics collection is disabled.

This design is critical for performance and code simplicity within the networking layer. By injecting this implementation, higher-level components like the packet encoder and decoder can unconditionally call methods like recordSend without the need for null checks or conditional logic (e.g., `if (statsRecorder != null)`). This avoids branching in performance-critical network I/O paths and simplifies the overall architecture.

It is the default, "silent" implementation used in scenarios where performance metrics are not required, such as production releases or specific server profiles.

### Lifecycle & Ownership
- **Creation:** Instantiated by the network bootstrap or a configuration factory when the feature flag for packet statistics is disabled. It is provided to components that require a PacketStatsRecorder interface.
- **Scope:** The lifetime of a NoopPacketStatsRecorder instance is typically tied to a network session or connection. However, as it is stateless, a single static instance could be shared across the entire application to reduce object allocation.
- **Destruction:** The object requires no explicit cleanup. It is eligible for garbage collection once the network session that holds a reference to it is terminated.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It does not store any information about the packets it is asked to record. The static EMPTY_ENTRY field is a constant, immutable object that provides zeroed-out default values.
- **Thread Safety:** The class is inherently **thread-safe**. Its methods are empty and it has no mutable state, making it safe to be called concurrently from any thread, including high-throughput network I/O threads managed by frameworks like Netty. No synchronization is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| recordSend(int, int, int) | void | O(1) | A no-operation method. It immediately returns without performing any action. |
| recordReceive(int, int, int) | void | O(1) | A no-operation method. It immediately returns without performing any action. |
| getEntry(int) | PacketStatsEntry | O(1) | Always returns the static, shared EMPTY_ENTRY instance, which contains zero values for all metrics. |

## Integration Patterns

### Standard Usage
A developer or system component should not interact with this class directly. Instead, it should be programmed against the PacketStatsRecorder interface. The framework is responsible for injecting the NoopPacketStatsRecorder when appropriate.

```java
// The network pipeline receives a PacketStatsRecorder, unaware of the concrete type.
// If stats are disabled, this 'recorder' instance will be a NoopPacketStatsRecorder.
PacketStatsRecorder recorder = networkSession.getStatsRecorder();

// This call does nothing and has negligible performance impact.
recorder.recordSend(packetId, uncompressedSize, compressedSize);
```

### Anti-Patterns (Do NOT do this)
- **Conditional Logic:** Do not write code that checks `instanceof NoopPacketStatsRecorder`. The purpose of this class is to eliminate such checks. Rely on the interface contract.
- **Expecting Metrics:** Do not call `getEntry` and expect meaningful data. Code that relies on statistics from the recorder must be prepared to handle the zeroed-out data returned by this implementation, which signifies that recording is disabled.

## Data Pipeline
This class acts as a data sink or a terminal node in the statistics pipeline. It accepts method calls but does not process, store, or forward any data.

> Flow:
> Packet Encoder/Decoder -> **NoopPacketStatsRecorder.recordSend()** -> (Data is discarded)
>
> Packet Encoder/Decoder -> **NoopPacketStatsRecorder.recordReceive()** -> (Data is discarded)

