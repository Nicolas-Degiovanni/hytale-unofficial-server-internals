---
description: Architectural reference for UpdateTime
---

# UpdateTime

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class UpdateTime implements Packet {
```

## Architecture & Concepts
The UpdateTime class is a network packet data structure, not a service or manager. Its sole purpose is to encapsulate and transport the state of the server's world time to the client. As a component of the Hytale network protocol, it acts as a contract for serializing and deserializing a specific piece of game state—the current in-game time—for transmission over the network.

This class is designed for efficiency and predictability. It has a fixed size and a well-defined binary layout, governed by the static constants within the class. It is a fundamental building block for synchronizing the client's world simulation with the server's authoritative state, ensuring a consistent player experience regarding time-of-day, lighting, and time-based game events.

## Lifecycle & Ownership
- **Creation:** On the server, an instance is created by the game loop or a dedicated time-of-day system to be sent to clients. On the client, an instance is created exclusively by the network protocol layer when a raw byte buffer with packet ID 146 is received and passed to the static `deserialize` factory method.
- **Scope:** Extremely short-lived and transient. An UpdateTime object exists only for the brief moment it takes to be serialized into a network buffer or deserialized from one and processed by a packet handler. It is a message, not a persistent entity.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by the target system, such as the client's World or Rendering engine. There is no persistent ownership of this object.

## Internal State & Concurrency
- **State:** Mutable. The internal state consists of a single nullable field, gameTime, which is an instance of InstantData. While technically mutable, instances of UpdateTime should be treated as immutable snapshots of data once created or deserialized.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms. It is designed to be created, processed, and discarded within a single thread context, typically a Netty network thread or the main game logic thread. Sharing an instance across multiple threads without external locking will lead to undefined behavior and is a critical anti-pattern.

## API Surface
The public API is primarily for use by the network protocol framework for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdateTime | O(1) | Constructs an UpdateTime object from a Netty ByteBuf. This is the primary entry point for packet creation on the client. |
| serialize(buf) | void | O(1) | Writes the object's state into a Netty ByteBuf. This is the primary exit point for sending the packet from the server. |
| computeSize() | int | O(1) | Returns the constant size of the packet in bytes, which is 13. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a preliminary check to ensure a buffer is large enough to contain a valid packet. |
| clone() | UpdateTime | O(1) | Creates a deep copy of the packet. Primarily used for defensive copying if the packet needs to be queued or processed later. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol stack. A packet handler receives the deserialized object, extracts the time data, and forwards it to the relevant game system.

```java
// Executed within a client-side network packet handler
void handlePacket(UpdateTime packet) {
    if (packet.gameTime != null) {
        WorldTimeManager timeManager = context.getService(WorldTimeManager.class);
        timeManager.setServerTime(packet.gameTime);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Deserialization:** Never use `new UpdateTime()` and manually populate its fields from a network buffer. This bypasses the correct null-checking and offset logic handled by the static `deserialize` method.
- **State Modification:** Do not modify the `gameTime` field after the packet has been deserialized. It represents a server-authoritative snapshot in time and should be treated as read-only.
- **Long-Term Storage:** Do not hold references to UpdateTime objects. Extract the internal InstantData and discard the packet object to allow for garbage collection.

## Data Pipeline
The flow of data for this packet is unidirectional from server to client. It is a core component of the state synchronization pipeline.

> **Flow:**
> Server World Clock -> `new UpdateTime(currentTime)` -> **UpdateTime.serialize()** -> Netty Channel -> Client Network Receiver -> **UpdateTime.deserialize()** -> Client Packet Handler -> Client WorldTimeManager -> Game State Update

