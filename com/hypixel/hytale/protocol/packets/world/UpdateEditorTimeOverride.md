---
description: Architectural reference for UpdateEditorTimeOverride
---

# UpdateEditorTimeOverride

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class UpdateEditorTimeOverride implements Packet {
```

## Architecture & Concepts
The UpdateEditorTimeOverride packet is a specialized network command used exclusively within the game's editor mode. It serves as a direct instruction from a server (or a host client) to a connected client, commanding a change to the client's local world time simulation. Architecturally, it is a fundamental component of the **Real-Time Editor State Synchronization** system.

Unlike continuous time-of-day synchronization packets used in normal gameplay, this packet represents a discrete, user-initiated override. It allows editor tools to pause, resume, or set the world time to a specific value, facilitating content creation and testing. Its design prioritizes low latency and predictable, fixed-size serialization for efficient handling by the network protocol engine.

The packet encapsulates two key pieces of state: an optional target game time and a mandatory pause flag. The use of a bitmask (the internal *nullBits* byte) for the optional time value is a common engine pattern to minimize payload size.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the server's editor logic in response to a privileged user action (e.g., interacting with a time control UI). The server populates its fields and enqueues it for transmission to a specific client or all clients in the editor session.
    - **Client-Side:** Instantiated by the client's network protocol layer when the corresponding packet ID (147) is read from the incoming network buffer. The static *deserialize* method acts as the factory.
- **Scope:** Transient and extremely short-lived. An instance of this packet exists only for the brief duration of its serialization, network transit, and subsequent deserialization and processing by a handler. It is not retained in any long-term state.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by the relevant client-side system (e.g., the World or TimeManager). There is no manual destruction or pooling mechanism for this packet type.

## Internal State & Concurrency
- **State:** Mutable. The packet is a simple container for its data fields, *gameTime* and *paused*. Its state is not intended to be modified after its initial creation and population.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created and written on a single thread (e.g., the server's main logic thread) and read on a single, different thread (e.g., a client's Netty worker thread). The network protocol framework guarantees the necessary memory visibility and safe handoff between threads.

## API Surface
The public contract is primarily for the protocol engine itself. Direct interaction with these methods outside the network layer is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UpdateEditorTimeOverride(gameTime, paused) | constructor | O(1) | Constructs a new packet with the specified time and pause state. |
| serialize(ByteBuf) | void | O(1) | Encodes the packet's state into a binary format. This is a fixed-size operation. |
| deserialize(ByteBuf, offset) | static UpdateEditorTimeOverride | O(1) | Decodes a packet from the given buffer. Acts as the client-side factory. |
| computeSize() | int | O(1) | Returns the exact, constant size of the packet payload in bytes (14). |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game logic developers. It is created and sent by the server's core editor systems and consumed by the client's packet handler registry.

A client-side handler would process the packet as follows:
```java
// Example of a client-side handler receiving the packet
public void handle(UpdateEditorTimeOverride packet) {
    World world = this.client.getWorld();
    TimeManager timeManager = world.getTimeManager();

    if (packet.gameTime != null) {
        timeManager.setOverrideTime(packet.gameTime.toGameTime());
    }

    timeManager.setPaused(packet.paused);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send the same packet instance. Each command must be a new object to prevent race conditions and ensure state integrity.
- **Manual Serialization:** Do not call *serialize* or *deserialize* directly. These methods are strictly for the internal protocol engine, which manages buffer allocation and lifecycle.
- **Client-Side Instantiation:** Clients should never create this packet using *new*. It is a server-to-client command only. Creating it on the client has no effect.

## Data Pipeline
The flow of this data is unidirectional from the server to the client.

> Flow:
> Server Editor Command -> Game Logic -> **new UpdateEditorTimeOverride()** -> Protocol Engine (serialize) -> Netty Channel -> Network -> Client Netty Channel -> Protocol Engine (deserialize) -> Client Packet Handler -> World TimeManager Update

