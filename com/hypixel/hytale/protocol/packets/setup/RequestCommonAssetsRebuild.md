---
description: Architectural reference for RequestCommonAssetsRebuild
---

# RequestCommonAssetsRebuild

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient

## Definition
```java
// Signature
public class RequestCommonAssetsRebuild implements Packet {
```

## Architecture & Concepts
The RequestCommonAssetsRebuild packet is a stateless **signal packet** used during the client-server setup phase. It functions as a command, not a data container, instructing the recipient to initiate a full rebuild of its common asset cache.

This packet is fundamentally a zero-payload message. Its entire meaning is conveyed by its Packet ID (28). The presence of this packet in the network stream is the command itself. This design is highly efficient for simple, unambiguous instructions, as it consumes minimal network bandwidth and requires trivial CPU cycles for serialization and deserialization.

It is typically sent by the server to the client under specific conditions, such as after a game update that invalidates existing client-side assets, or to force a client to repair a potentially corrupted asset cache. The operation it triggers is heavyweight and I/O intensive, and should not be requested frequently.

### Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1. **Inbound:** The protocol's packet dispatcher instantiates it via the static `deserialize` factory method when an incoming network buffer is identified with Packet ID 28.
    2. **Outbound:** Game logic on the sending side (typically the server) creates a new instance (`new RequestCommonAssetsRebuild()`) before passing it to the network layer for serialization.
- **Scope:** This object is extremely short-lived. It exists only for the duration of its handling within a single network event loop tick. It is a pure message DTO (Data Transfer Object) and should be considered expired immediately after its corresponding handler has been invoked.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for collection as soon as it falls out of the scope of the packet handler. No manual resource management is necessary.

## Internal State & Concurrency
- **State:** This object is **stateless and immutable**. It contains no instance fields. All behavior is derived from its type and static constants.
- **Thread Safety:** The class is inherently thread-safe. An instance can be safely passed between threads without synchronization.
    - **Warning:** While the packet itself is thread-safe, the system that *processes* it (e.g., an AssetManager) is almost certainly not. The action triggered by this packet—rebuilding assets—is a long-running, I/O-bound task that must be dispatched from the network I/O thread to a dedicated worker thread or the main game thread to prevent blocking network operations.

## API Surface
The public API is minimal, reflecting the packet's role as a simple signal.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type, which is 28. |
| serialize(ByteBuf) | void | O(1) | Writes zero bytes to the buffer. This packet has no payload. |
| deserialize(ByteBuf, int) | RequestCommonAssetsRebuild | O(1) | Static factory. Reads zero bytes and returns a new singleton-like instance. |

## Integration Patterns

### Standard Usage
This packet is processed by a handler that is registered with the network protocol dispatcher. The handler's sole responsibility is to delegate the asset rebuild task to the appropriate system.

```java
// Example within a central PacketHandler
public void processPacket(Packet packet) {
    if (packet instanceof RequestCommonAssetsRebuild) {
        // WARNING: This is a significant, blocking operation.
        // It must be scheduled and not executed directly on the network thread.
        AssetService assetService = context.getService(AssetService.class);
        assetService.scheduleCommonAssetRebuild();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not store an instance of RequestCommonAssetsRebuild in a field or collection. It is a transient message, not state. Caching or reusing it serves no purpose and can complicate debugging.
- **Direct Execution on I/O Thread:** Never invoke the asset rebuilding logic directly from the packet handler that runs on a Netty I/O thread. Doing so will block all network communication for the duration of the rebuild, likely causing a client timeout and disconnect.

## Data Pipeline
The data flow for this packet is trivial, as it contains no payload. The primary flow is the command's journey from the server's intent to the client's action.

> Flow (Server to Client):
> Server Logic -> `new RequestCommonAssetsRebuild()` -> Network Encoder -> TCP/IP Stream -> Client Netty Pipeline -> Packet Frame Decoder -> Packet Dispatcher (ID 28) -> **RequestCommonAssetsRebuild** instance -> Packet Handler -> AssetService Scheduler

