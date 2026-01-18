---
description: Architectural reference for ServerSetPaused
---

# ServerSetPaused

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class ServerSetPaused implements Packet {
```

## Architecture & Concepts
The ServerSetPaused class is a network packet data structure, not a service or manager. Its sole purpose is to encapsulate and transport a single boolean state from the server to the client, indicating whether the game simulation should be paused or resumed.

As part of the Hytale Protocol, it is a server-authoritative command. It fits within the "control plane" of the networking layer, responsible for managing game state rather than transmitting bulk data like world chunks or entity positions. The packet's design emphasizes extreme efficiency, with a fixed size of one byte and no variable fields, ensuring minimal overhead for this frequent and critical state change.

The static constants defined within the class, such as PACKET_ID and FIXED_BLOCK_SIZE, are metadata used by the higher-level protocol codec to identify, validate, and route the incoming byte stream to the correct deserializer.

### Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated on-demand by the server's game logic when a global pause or resume event occurs. For example, an administrator command or the start of a scripted cinematic sequence would trigger its creation.
    - **Client-Side:** Instantiated exclusively by the network protocol decoder when a raw network buffer containing the packet ID 159 is received. The static `deserialize` method acts as the factory.

- **Scope:** Ephemeral. An instance of ServerSetPaused exists only for the brief duration of its serialization, network transit, and subsequent deserialization and handling. It is a fire-and-forget message.

- **Destruction:** The object is immediately eligible for garbage collection after its `paused` field has been read by the client-side packet handler. It is not cached or retained by any system.

## Internal State & Concurrency
- **State:** The internal state consists of a single, mutable boolean field named `paused`. The object is a simple data container with no complex internal logic.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, written, and read within a single-threaded context, such as a server's main tick loop or a client's Netty event loop. No locking mechanisms are implemented. Accessing an instance from multiple threads without external synchronization will lead to undefined behavior.

## API Surface
The public API is designed for interaction with the protocol serialization pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (159). |
| serialize(ByteBuf) | void | O(1) | Writes the boolean state as a single byte into the provided buffer. |
| deserialize(ByteBuf, int) | ServerSetPaused | O(1) | **Static Factory.** Reads one byte from the buffer and constructs a new ServerSetPaused instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload in bytes (1). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game feature developers. It is created and consumed by the core engine components.

**Server-Side (Conceptual): Sending the packet**
```java
// In a server-side system managing game state
// This would be wrapped in a higher-level network manager API
void pauseGameForAllPlayers(boolean shouldPause) {
    ServerSetPaused pausePacket = new ServerSetPaused(shouldPause);
    networkManager.broadcast(pausePacket);
}
```

**Client-Side (Conceptual): Handling the packet**
```java
// In a client-side packet handler
public void handle(ServerSetPaused packet) {
    // The game's simulation or update loop will check this state
    this.gameOrchestrator.setSimulationPaused(packet.paused);
}
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** A client must never create an instance of ServerSetPaused to send to the server. This is a server-to-client packet only. Attempting to send it from a client is a protocol violation and will be rejected by the server.
- **State Caching:** Do not hold a reference to a received ServerSetPaused packet. Its value represents a point-in-time state. Copy the `paused` boolean to the relevant game state manager and discard the packet object.
- **Manual Serialization:** Developers should never call `serialize` or `deserialize` directly. These methods are exclusively invoked by the engine's network codec (e.g., a Netty ChannelInboundHandler).

## Data Pipeline
The flow of data for this packet is unidirectional from server to client.

> **Flow:**
> Server Game State Change → `new ServerSetPaused()` → Protocol Encoder → **ServerSetPaused.serialize()** → Network Buffer → Client Network Stack → Protocol Decoder → **ServerSetPaused.deserialize()** → Client Packet Handler → Client Game State Update

