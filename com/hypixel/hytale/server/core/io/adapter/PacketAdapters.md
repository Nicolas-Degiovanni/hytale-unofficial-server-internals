---
description: Architectural reference for PacketAdapters
---

# PacketAdapters

**Package:** com.hypixel.hytale.server.core.io.adapter
**Type:** Static Utility / Singleton

## Definition
```java
// Signature
public class PacketAdapters {
```

## Architecture & Concepts
The PacketAdapters class is a central, static hub for intercepting, inspecting, and potentially consuming network packets. It implements the **Interceptor** and **Chain of Responsibility** design patterns, providing a powerful mechanism to decouple core network processing from auxiliary systems like logging, analytics, or security modules.

This class acts as a global middleware layer for the server's entire packet I/O pipeline. It maintains two distinct, ordered chains of handlers: one for **inbound** packets (Client to Server) and one for **outbound** packets (Server to Client).

When a packet is processed, it is passed through the appropriate chain of registered PacketFilter instances. Each filter can inspect the packet and its associated connection handler. A filter can choose to "consume" the packet by returning true, which immediately halts further processing of that packet down the chain, preventing it from reaching its default handler. If all filters return false, the packet proceeds to its intended destination.

This architecture allows for dynamic modification of the server's network behavior at runtime without altering the core network stack.

### Lifecycle & Ownership
- **Creation:** As a static class, PacketAdapters is not instantiated. Its internal state, the handler lists, is initialized by the JVM class loader when the class is first referenced. This typically occurs during server bootstrap.
- **Scope:** The scope is global and application-wide. The registered inbound and outbound handlers persist for the entire lifetime of the server process.
- **Destruction:** The handler lists are only cleared when the JVM process terminates.

**WARNING:** Because handlers are stored in a global static list, they can easily cause memory leaks. Any component that registers a handler (e.g., a game mode module, a temporary debugging tool) **must** explicitly call the corresponding deregister method during its own shutdown sequence to prevent its objects from being retained indefinitely.

## Internal State & Concurrency
- **State:** The class maintains two primary pieces of mutable state: the static `inboundHandlers` list and the static `outboundHandlers` list. These lists are modified at runtime via the public register and deregister methods.
- **Thread Safety:** This class is **thread-safe**. It achieves this by using CopyOnWriteArrayList for its handler lists.
    - **Reads:** Iterating the handlers (during `__handleInbound` or `__handleOutbound`) is extremely fast and lock-free, as it operates on an immutable snapshot of the list at the time the iteration began. This is critical for high-performance network I/O threads.
    - **Writes:** Adding or removing a handler (`register...`, `deregister...`) is an expensive operation. It involves creating a complete copy of the underlying array. This design choice is optimized for "read-mostly, write-rarely" scenarios, which is the expected usage pattern for packet interceptors.

**WARNING:** While thread-safe, frequent registration and deregistration of handlers can lead to performance degradation and memory pressure due to the copy-on-write mechanism. Handlers should ideally be registered once during component initialization.

## API Surface
The primary API surface consists of methods for registering and deregistering handlers. The `__handle` methods are considered internal to the framework and should not be called directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerInbound(PacketFilter) | void | O(N) | Registers a filter to intercept inbound packets. The cost is linear to the number of existing handlers. |
| registerOutbound(PacketFilter) | void | O(N) | Registers a filter to intercept outbound packets. |
| deregisterInbound(PacketFilter) | void | O(N) | Removes a previously registered inbound filter. Throws IllegalArgumentException if the filter was not found. |
| deregisterOutbound(PacketFilter) | void | O(N) | Removes a previously registered outbound filter. Throws IllegalArgumentException if the filter was not found. |
| registerInbound(PacketWatcher) | PacketFilter | O(N) | A convenience method to register a read-only "watcher" that cannot consume the packet. |

## Integration Patterns

### Standard Usage
The most common use case is to register a non-consuming watcher to log or analyze specific packets for a player.

```java
// Register a watcher to log all inbound GamePacket instances for a specific player
PlayerPacketWatcher myWatcher = (player, packet) -> {
    if (packet instanceof GamePacket) {
        HytaleLogger.info("Player " + player.getName() + " sent packet: " + packet.getClass().getSimpleName());
    }
};

// The returned filter must be stored to be able to deregister it later
PacketFilter registeredFilter = PacketAdapters.registerInbound(myWatcher);

// ... later, during cleanup ...
try {
    PacketAdapters.deregisterInbound(registeredFilter);
} catch (IllegalArgumentException e) {
    // Handle cases where it might have already been deregistered
}
```

### Anti-Patterns (Do NOT do this)
- **Leaking Handlers:** Do not register a handler and fail to store the returned PacketFilter object. Without this reference, you cannot deregister the handler, leading to a guaranteed memory leak.
- **Blocking Operations:** Do not perform blocking I/O, database queries, or long-running computations inside a filter. Filters are executed synchronously on the server's network I/O threads. Any delay will directly increase server latency and reduce overall throughput. Offload heavy work to a separate thread pool.
- **Ignoring Return Value:** When implementing a PacketFilter directly (not a watcher), be deliberate about the boolean return value. Returning true can have significant side effects by preventing other systems and the core game logic from ever seeing the packet.

## Data Pipeline
PacketAdapters is positioned early in the data flow, immediately after deserialization for inbound packets and just before serialization for outbound packets.

> **Inbound Flow:**
> Network Socket -> Byte Decoder -> Packet Deserializer -> **PacketAdapters.__handleInbound** -> [Filter Chain] -> Core PacketHandler (e.g., GamePacketHandler)

> **Outbound Flow:**
> Core Logic (e.g., Game System) -> **PacketAdapters.__handleOutbound** -> [Filter Chain] -> Packet Serializer -> Byte Encoder -> Network Socket

