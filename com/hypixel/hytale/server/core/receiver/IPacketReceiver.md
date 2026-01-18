---
description: Architectural reference for IPacketReceiver
---

# IPacketReceiver

**Package:** com.hypixel.hytale.server.core.receiver
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IPacketReceiver {
```

## Architecture & Concepts
The IPacketReceiver interface defines the fundamental contract for any object capable of receiving and dispatching outgoing network packets on the server. It serves as a critical abstraction layer, decoupling high-level game logic from the low-level details of network transport and connection management.

In the server architecture, an IPacketReceiver typically represents the network endpoint for a single client connection. Systems that need to send data to a client—such as the World State Synchronizer, Chat System, or Entity Manager—do not interact directly with network sockets or channels. Instead, they acquire a reference to the target client's IPacketReceiver implementation and use its methods to queue packets for delivery.

This design pattern centralizes the responsibility of packet serialization, batching, and transport into concrete implementations, while providing a clean, high-level API for the rest of the server's systems.

### Lifecycle & Ownership
As an interface, IPacketReceiver does not have its own lifecycle. The lifecycle is dictated entirely by the concrete class that implements it, which is almost always a session or connection management object.

- **Creation:** An object implementing IPacketReceiver is instantiated when a client successfully establishes a connection with the server. This is typically handled by a network connection listener or a session factory.
- **Scope:** The instance exists for the duration of a client's network session. It is intrinsically tied to a single, active connection.
- **Destruction:** The instance is marked for garbage collection when the associated client connection is terminated, either gracefully (logout) or forcefully (timeout, network error). Any remaining references to a destroyed receiver will likely result in exceptions or silent failures.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any non-trivial implementation is expected to be stateful. Implementations will manage internal buffers, packet queues, or references to the underlying network channel (e.g., a Netty Channel).
- **Thread Safety:** **Implementations of this interface must be thread-safe.** Server systems operating on different threads (e.g., main game tick, physics, AI) will concurrently call write methods to send data to the same client. Implementations are expected to use appropriate concurrency controls, such as synchronized blocks, concurrent queues, or confinement to a single network I/O thread, to prevent race conditions and packet corruption.

## API Surface
The public contract is minimal, focusing exclusively on the action of sending a packet. The distinction between the two methods is critical for performance and delivery guarantees.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| write(Packet) | void | O(1) | Queues a packet for delivery. This is the standard method and may involve batching or caching for network efficiency. |
| writeNoCache(Packet) | void | O(1) | Queues a packet for immediate or high-priority delivery, bypassing any standard batching mechanisms. |

**WARNING:** The use of writeNoCache should be reserved for latency-sensitive packets where the overhead of bypassing the cache is justified. Overuse can degrade overall network performance.

## Integration Patterns

### Standard Usage
Game logic should retrieve the IPacketReceiver for a specific player or connection from a central registry or session object and use it to dispatch packets.

```java
// Example: A system sending a health update to a player
Player targetPlayer = getPlayerById(playerId);
IPacketReceiver connection = targetPlayer.getConnection();

HealthUpdatePacket healthPacket = new HealthUpdatePacket(targetPlayer.getHealth());
connection.write(healthPacket);
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived References:** Do not cache an IPacketReceiver instance in a long-lived service. The connection it represents is ephemeral. Always re-acquire the receiver from the player or session object before use to ensure the connection is still valid.
- **Ignoring Thread Safety:** Do not assume the underlying implementation is non-blocking. While typical, it is not guaranteed by the contract. Sending a large volume of packets from a performance-critical thread (like the main server tick) without understanding the implementation's performance characteristics can lead to server stalls.
- **Misusing writeNoCache:** Do not use writeNoCache for routine, high-frequency packets like entity position updates. This will flood the network transport, defeating the purpose of send-side batching and potentially increasing latency.

## Data Pipeline
IPacketReceiver acts as the entry point or "sink" for the server's outgoing data pipeline. It is the bridge between abstract game data and the network stack.

> Flow:
> Game System Logic -> Packet Object Instantiation -> **IPacketReceiver.write()** -> Internal Packet Queue -> Network Encoder -> TCP/UDP Socket

