---
description: Architectural reference for UpdateServerAccess
---

# UpdateServerAccess

**Package:** com.hypixel.hytale.protocol.packets.serveraccess
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class UpdateServerAccess implements Packet {
```

## Architecture & Concepts
The UpdateServerAccess class is a Data Transfer Object (DTO) that represents a specific message within the Hytale network protocol. It does not contain business logic; its sole purpose is to structure and carry data for transmission between the server and client.

This packet is part of the Server Access protocol flow, used by a server to inform a client about changes to its accessibility status. For example, it can signal a change from a public server to a private, friends-only server. It also provides an updated list of host addresses that the client may need to connect to.

As a concrete implementation of the Packet interface, it adheres to a strict contract for serialization, deserialization, and size computation, allowing it to be processed generically by the engine's network pipeline. The serialization format is highly optimized, using bit fields for nullability checks and variable-length integers to minimize payload size.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1. **Inbound:** The network protocol decoder instantiates the object by calling the static `deserialize` method when an incoming data buffer with Packet ID 251 is identified.
    2. **Outbound:** Server-side game logic creates an instance via its constructor when it needs to notify a client of a change in server access policy.
- **Scope:** The object is ephemeral and has a very short lifetime. It exists only for the duration of a single network event processing cycle. Once its data has been read by a handler or written to a network buffer, it is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Cleanup is managed exclusively by the Java Garbage Collector. There are no manual resource management requirements.

## Internal State & Concurrency
- **State:** The class is a mutable data holder. Its fields, `access` and `hosts`, can be modified after construction. It does not perform any caching or maintain state beyond the data it is designed to transport.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread or the main game thread.
    - **WARNING:** Sharing an instance of UpdateServerAccess across multiple threads without explicit, external synchronization will lead to race conditions and unpredictable behavior. Do not store instances of this packet in shared state.

## API Surface
The public contract is dominated by static methods used by the protocol framework for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | UpdateServerAccess | O(N) | **[Entry Point]** Constructs a new object by reading from a ByteBuf. N is the number of hosts. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. N is the number of hosts. |
| computeSize() | int | O(N) | Calculates the exact byte size the serialized packet will occupy. N is the number of hosts. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a lightweight check on a buffer to ensure it contains a valid packet structure without full deserialization. |
| clone() | UpdateServerAccess | O(N) | Creates a deep copy of the packet and its contained data. |

## Integration Patterns

### Standard Usage
This packet is almost exclusively handled by the network layer and packet listeners. A developer would typically interact with it inside a handler that receives the deserialized object.

```java
// Example of a packet handler receiving the object
public void handle(UpdateServerAccess packet) {
    ServerInfo currentServer = this.serverManager.getCurrentServer();
    currentServer.setAccess(packet.access);
    currentServer.updateHosts(packet.hosts);
    
    // The 'packet' object is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not hold a reference to a packet object after its initial processing. They are meant to be single-use DTOs.
- **Cross-Thread Modification:** Do not create a packet on one thread and pass it to the network layer on another thread if modifications can still occur. Finalize its state before handing it off.
- **Manual Deserialization:** Do not call `deserialize` directly. This is the responsibility of the protocol's packet dispatcher, which manages buffer offsets and packet IDs.

## Data Pipeline
The UpdateServerAccess packet is a data container that moves through the network stack.

**Inbound Flow (Server to Client):**
> Flow:
> Raw TCP Bytes -> Netty ByteBuf -> Protocol Decoder -> **UpdateServerAccess.deserialize** -> Packet Handler -> Game State Update

**Outbound Flow (Server to Client):**
> Flow:
> Game Logic Event -> `new UpdateServerAccess()` -> **UpdateServerAccess.serialize** -> Protocol Encoder -> Netty ByteBuf -> Raw TCP Bytes

