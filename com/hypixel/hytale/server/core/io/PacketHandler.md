---
description: Architectural reference for PacketHandler
---

# PacketHandler

**Package:** com.hypixel.hytale.server.core.io
**Type:** Transient

## Definition
```java
// Signature
public abstract class PacketHandler implements IPacketReceiver {
```

## Architecture & Concepts
The PacketHandler is an abstract base class that forms the cornerstone of the server's network communication layer for a single client connection. It embodies the **State pattern**, where different concrete implementations represent distinct phases of a client's session, such as authentication, gameplay, or status querying.

This class acts as the primary bridge between the low-level Netty network pipeline and the server's high-level game logic. Its responsibilities include processing inbound packets, dispatching outbound packets, managing connection state, and handling session lifecycle events like keep-alives (pings), timeouts, and disconnection.

Each active client connection is associated with a Netty Channel, and that Channel's pipeline will contain exactly one active PacketHandler instance at any given time. The server transitions a client between states (e.g., from login to gameplay) by replacing the current PacketHandler in the pipeline with a new one.

### Lifecycle & Ownership
- **Creation:** A concrete PacketHandler is instantiated by the server's network bootstrap logic (e.g., a Netty ChannelInitializer) when a new connection is accepted. The initial handler type depends on the entry point, but a typical flow starts with a handler responsible for the initial handshake and version negotiation.
- **Scope:** The lifetime of a specific PacketHandler instance is transient and corresponds to a single state in the connection's lifecycle. For example, an AuthenticationPacketHandler exists only until the player is successfully authenticated, at which point it is replaced by a GamePacketHandler. The overall connection, managed by the underlying Netty Channel, persists across these handler swaps.
- **Destruction:** An instance is eligible for garbage collection after it has been replaced by a new handler in the pipeline (via the `unregistered` callback) or when the underlying network channel is closed. The `closed` method provides a final hook for cleanup before the connection is terminated.

## Internal State & Concurrency
- **State:** The PacketHandler is highly stateful and mutable. It maintains critical connection-specific data, including the associated Netty Channel, the negotiated ProtocolVersion, player authentication details, and extensive ping/latency metrics. It also manages a scheduled `timeoutTask` to prevent stalled connections.

- **Thread Safety:** **CRITICAL WARNING:** This class is not thread-safe and is designed to be operated exclusively by a single Netty EventLoop thread. All packet handling (`handle`), writing (`write`), and lifecycle methods (`registered`, `unregistered`, `closed`) are invoked by the EventLoop. Off-thread access will lead to race conditions, packet corruption, and unpredictable connection state.

  The nested PingInfo class utilizes `ReentrantLock` to protect its internal metric collections. This suggests that metric data may be read by a separate monitoring or diagnostics thread, but all mutations must still originate from the EventLoop thread during packet processing.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(Packet packet) | void | O(1) | Primary entry point for processing a deserialized inbound packet. |
| accept(Packet packet) | void | - | Abstract method implemented by subclasses to define state-specific packet logic. |
| write(Packet... packets) | void | O(N) | Writes one or more packets to the outbound channel buffer. May flush immediately or queue based on internal state. |
| disconnect(String message) | void | O(1) | Initiates a graceful disconnection, sending a final Disconnect packet to the client before closing the channel. |
| registered(PacketHandler old) | void | O(1) | Lifecycle callback invoked by the Netty pipeline when this handler becomes active. |
| unregistered(PacketHandler new) | void | O(1) | Lifecycle callback invoked by the Netty pipeline when this handler is being replaced or removed. |
| setTimeout(String stage, ...) | void | O(1) | Schedules a task to disconnect the client if a condition is not met within a specified duration. Essential for preventing stalled logins. |
| clearTimeout() | void | O(1) | Cancels any pending timeout task. Must be called upon successful state transition. |
| tickPing(float dt) | void | O(1) | Called by the server tick loop to manage the keep-alive mechanism and send periodic Ping packets. |

## Integration Patterns

### Standard Usage
The PacketHandler is not used directly but is extended to create concrete state handlers. The server's core logic interacts with the handler primarily to send packets or to initiate a disconnect. It does not manage the handler's lifecycle, which is delegated to the Netty pipeline.

```java
// Example of a concrete handler processing a packet
public class GamePacketHandler extends PacketHandler {
    // ... constructor ...

    @Override
    public void accept(@Nonnull Packet packet) {
        if (packet instanceof PlayerActionPacket) {
            // Process game logic
        } else if (packet instanceof ChatPacket) {
            // Broadcast chat message
        }
    }

    public void sendWorldData(WorldData data) {
        // Sending a packet from server logic
        write(new WorldDataPacket(data));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually create a PacketHandler with `new`. Handlers are state objects managed exclusively by the server's Netty ChannelPipeline. Manipulating the pipeline directly is the only correct way to transition connection states.
- **Cross-Thread Writing:** Do not call `write` or `disconnect` from a thread other than the connection's assigned Netty EventLoop. This will violate Netty's threading model and cause severe, difficult-to-debug issues. If an action from another thread must send a packet, it must schedule a task on the correct EventLoop.
- **Ignoring Timeouts:** Failure to call `clearTimeout` after a successful state transition (e.g., post-login) will result in the player being incorrectly disconnected by the timeout task from the previous state. Conversely, failing to call `setTimeout` at the beginning of a sensitive stage (like authentication) can leave the server vulnerable to slow-loris style attacks.

## Data Pipeline
The PacketHandler is the central processing component in the server's data flow for a single connection.

> **Inbound Flow:**
> Netty Channel -> Byte Decryption/Decompression -> Packet Deserializer -> **PacketHandler.handle()** -> Concrete Handler Logic -> Server Systems

> **Outbound Flow:**
> Server Systems -> **PacketHandler.write()** -> Packet Caching -> Packet Serializer -> Byte Encryption/Compression -> Netty Channel

