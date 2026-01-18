---
description: Architectural reference for WorldLoadFinished
---

# WorldLoadFinished

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class WorldLoadFinished implements Packet {
```

## Architecture & Concepts
The WorldLoadFinished packet is a stateless **signal packet** used within the connection setup protocol. Its primary architectural role is to act as a synchronization event, notifying a remote endpoint that all necessary world data has been loaded and processed, and the local client is ready to proceed to the next stage of the game loop, such as rendering the world and enabling player controls.

This packet is fundamentally a marker. It carries no payload, and its entire meaning is conveyed by its type, identified by its static PACKET_ID. Its presence in the network stream serves as a trigger for a state transition in the connection's state machine. For example, upon receiving this packet, a server might begin sending entity updates, or a client might remove a loading screen.

Due to its zero-byte payload, it is an extremely lightweight and efficient mechanism for coordinating state between the client and server during the critical world-entry phase.

## Lifecycle & Ownership
- **Creation:** An instance of WorldLoadFinished is created under two circumstances:
    1.  **Inbound:** By the protocol's deserialization layer when a packet with ID 22 is read from the network buffer.
    2.  **Outbound:** By high-level game logic (e.g., a WorldManager) that needs to signal its readiness to the remote endpoint.
- **Scope:** Transient and extremely short-lived. An instance exists only for the duration of its processing by a single network event handler. It is not designed to be cached or held by any long-term service.
- **Destruction:** The object becomes eligible for garbage collection immediately after the packet handler that received it completes its execution. There are no persistent references to it.

## Internal State & Concurrency
- **State:** **Immutable**. The WorldLoadFinished class contains no instance fields and therefore has no internal state. All instances are functionally identical and interchangeable. The class definition consists solely of static constants that define the packet's metadata for the protocol framework.
- **Thread Safety:** **Inherently thread-safe**. As a stateless, immutable object, it can be safely shared and read across multiple threads without any risk of data corruption. In practice, it is almost always handled exclusively by a single Netty I/O worker thread.

## API Surface
The public API is designed for consumption by the network protocol framework, not for direct manipulation by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static packet identifier (22). |
| serialize(ByteBuf) | void | O(1) | Writes zero bytes to the buffer. This packet has no payload. |
| deserialize(ByteBuf, int) | WorldLoadFinished | O(1) | Creates a new instance without reading from the buffer. |
| clone() | WorldLoadFinished | O(1) | Returns a new, identical instance of the packet. |

## Integration Patterns

### Standard Usage
Developers should not instantiate or serialize this packet directly. Interaction is typically limited to receiving it within a designated packet handler to trigger a change in game state.

```java
// A handler class listens for specific packet types.
// The framework routes the deserialized packet here.
public class ClientSetupPacketHandler {

    private final GameStateManager gameStateManager;

    public void handle(WorldLoadFinished packet) {
        // The receipt of this packet is the signal.
        // No data is read from the packet itself.
        this.gameStateManager.transitionTo(State.IN_GAME);
        this.gameStateManager.unlockPlayerInput();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Adding Fields:** Do not modify this class to include data. Its contract is to be a payload-less signal. If state needs to be transmitted, a new and distinct packet type must be created.
- **Expecting a Payload:** Do not attempt to read data from the ByteBuf when deserializing this packet. Its size is guaranteed to be zero.
- **Redundant Sending:** Sending this packet multiple times within the same connection phase is unnecessary and may indicate a logic error in the state machine.

## Data Pipeline
The "data" in this pipeline is the event of the packet's arrival itself.

> **Outbound Flow (Client to Server):**
> Client WorldManager -> `Connection.send(new WorldLoadFinished())` -> Protocol Encoder -> **WorldLoadFinished.serialize()** -> Netty Channel -> Network

> **Inbound Flow (Server receives from Client):**
> Network -> Netty Channel -> Protocol Decoder -> **WorldLoadFinished.deserialize()** -> Server Packet Handler -> Update Player State

