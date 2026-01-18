---
description: Architectural reference for ClientReady
---

# ClientReady

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ClientReady implements Packet {
```

## Architecture & Concepts
The ClientReady packet is a fundamental signaling mechanism within the Hytale network protocol. It serves as a state transition message sent from the client to the server during the final stages of the connection and world-loading sequence.

Architecturally, this packet acts as a gate. The server will not begin streaming world data (chunks) or transition the player into the active gameplay state until it receives this specific packet. Its purpose is to ensure the client has finished its local loading processes—such as asset hydration and initial scene setup—before being inundated with real-time world information.

This class is a pure data container, embodying the Packet interface. It does not contain any logic beyond serialization and deserialization. The protocol framework relies on the static constants within this class (like PACKET_ID and FIXED_BLOCK_SIZE) for efficient and correct dispatching and parsing within the Netty pipeline.

### Lifecycle & Ownership
- **Creation:** An instance of ClientReady is created by a high-level client-side manager (e.g., a ConnectionManager or WorldLoadManager) once all prerequisite loading tasks are complete. It is a short-lived object.
- **Scope:** The object's lifecycle is bound to a single network transaction. On the client, it exists only long enough to be serialized into a network buffer. On the server, it is deserialized, processed by a single packet handler, and then becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction. The object is released for garbage collection immediately after its data has been used for serialization or business logic processing.

## Internal State & Concurrency
- **State:** The internal state is minimal and fully mutable, consisting of two boolean flags: readyForChunks and readyForGameplay. It holds no cached data or complex object graphs.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data structure intended to be created, populated, and processed within a single thread of execution, typically a Netty event loop thread. Concurrent modification from multiple threads will lead to unpredictable behavior and is a severe anti-pattern. The network protocol framework guarantees safe handoff between threads.

## API Surface
The public API is designed for interaction with the protocol framework, not for general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ClientReady(boolean, boolean) | constructor | O(1) | Constructs a new packet with specified readiness flags. |
| getId() | int | O(1) | Returns the static packet identifier (105). Used for protocol dispatching. |
| serialize(ByteBuf) | void | O(1) | Writes the two boolean flags into the provided network buffer. |
| deserialize(ByteBuf, int) | static ClientReady | O(1) | Reads two bytes from the buffer and constructs a new ClientReady instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a low-level check to ensure the buffer contains enough data for a valid packet. |

## Integration Patterns

### Standard Usage
The ClientReady packet is instantiated and sent by the client to signal its readiness to the server. The server receives it and updates the player's session state accordingly.

```java
// Client-side: After loading is complete
Connection connection = getClientConnection();
ClientReady readyPacket = new ClientReady(true, true);
connection.sendPacket(readyPacket);

// Server-side: Within a packet handler
public void handle(ClientReady packet) {
    PlayerSession session = getPlayerSession();
    if (packet.readyForChunks) {
        session.setReadyForWorldData(true);
        worldStreamingService.beginStreamingTo(session);
    }
    if (packet.readyForGameplay) {
        session.setReadyForGameplay(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Premature Sending:** Do not send this packet before the client has fully loaded all necessary assets. Doing so will cause the server to stream world data that the client cannot render, likely resulting in visual artifacts or crashes.
- **Direct Serialization:** Do not call the serialize or deserialize methods directly. These are low-level operations intended to be invoked exclusively by the network protocol engine.
- **State Reuse:** Do not hold onto and reuse a ClientReady instance. They are extremely lightweight and should be instantiated as needed for clarity and to avoid side effects.

## Data Pipeline
The flow of this packet represents a critical synchronization point between the client and server.

> **Flow:**
> Client Asset Manager -> **ClientReady** (Instantiation) -> Network Service -> Serialization Engine -> Netty ByteBuf -> Server Network Listener -> Packet Deserializer -> **ClientReady** (Re-hydrated) -> Server Packet Handler -> Player State Update -> World Streaming Service Activation

