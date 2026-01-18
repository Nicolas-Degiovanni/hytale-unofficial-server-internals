---
description: Architectural reference for PacketWatcher
---

# PacketWatcher

**Package:** com.hypixel.hytale.server.core.io.adapter
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface PacketWatcher extends BiConsumer<PacketHandler, Packet> {
```

## Architecture & Concepts
The PacketWatcher interface defines a behavioral contract for components that need to passively observe network packets flowing through the server's I/O pipeline. It is a specialized BiConsumer designed for network interception, acting as a tap or a hook into the stream of incoming data.

This interface embodies the **Observer** design pattern. It allows for the decoupling of core packet processing logic from cross-cutting concerns such as logging, metrics collection, or real-time debugging. Systems can register a PacketWatcher implementation with the network layer to be notified of every packet that a specific PacketHandler receives, without modifying the handler itself.

Its primary architectural role is to provide a standardized, non-invasive mechanism for monitoring server network traffic at the packet level.

### Lifecycle & Ownership
- **Creation:** PacketWatcher instances are almost exclusively created as lambda expressions or method references at the point of registration. They are not meant to be managed as standalone, stateful objects. The component responsible for a specific concern (e.g., a MetricsService) creates and registers the watcher with the server's network pipeline.
- **Scope:** The lifecycle of a PacketWatcher is bound to its registration. It exists only as long as it is held as a listener by the network pipeline component.
- **Destruction:** An instance is eligible for garbage collection once it is de-registered from the pipeline and no other references to it exist. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** As a functional interface, PacketWatcher itself is stateless. However, the implementing lambda can be stateful if it captures variables from its enclosing scope (i.e., it is a closure).

- **Thread Safety:** The interface contract is thread-safe. However, the implementation of the `accept` method is **not** guaranteed to be. It will be invoked directly by the server's network I/O threads (e.g., Netty worker threads). Therefore, any implementation of PacketWatcher **must be thread-safe**. Any shared state accessed or modified within the watcher must be protected by appropriate synchronization mechanisms to prevent race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(PacketHandler, Packet) | void | O(N) | Invoked by the network pipeline for an incoming packet. The complexity is dependent on the implementation (N). |

## Integration Patterns

### Standard Usage
A PacketWatcher is typically implemented as a lambda and registered with a central network management service to monitor traffic. This is the canonical pattern for instrumenting the network layer.

```java
// A metrics service registering a watcher to count packet types
NetworkPipeline pipeline = server.getNetworkPipeline();

pipeline.addGlobalWatcher((handler, packet) -> {
    // This code executes on a network I/O thread
    metrics.increment("packets.received." + packet.getClass().getSimpleName());
});
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** The `accept` method is executed on a critical network I/O thread. Performing any blocking operations, such as file I/O, database queries, or complex computations, will stall the thread. This can severely degrade or completely halt the server's ability to process network traffic. All operations must be non-blocking and extremely fast.

- **Unsynchronized State Modification:** Implementing a watcher that modifies shared, mutable state without proper synchronization is a severe anti-pattern. Since watchers are invoked by multiple I/O threads, this will inevitably lead to data corruption and race conditions.

- **Throwing Exceptions:** Unhandled exceptions thrown from a PacketWatcher implementation may propagate up the call stack and disrupt or terminate the network I/O thread, potentially crashing the server. All logic must be wrapped in robust error handling.

## Data Pipeline
The PacketWatcher is an observer within the data pipeline, not a transformative step. It receives data for inspection but is not expected to modify the packet or alter the primary flow.

> Flow:
> Network I/O Read -> Byte-to-Packet Decoder -> Network Pipeline -> **PacketWatcher.accept()** -> Primary PacketHandler Logic

