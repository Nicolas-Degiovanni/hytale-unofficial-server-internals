---
description: Architectural reference for RemoveMapMarker
---

# RemoveMapMarker

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class RemoveMapMarker implements Packet {
```

## Architecture & Concepts
The RemoveMapMarker class is a Data Transfer Object (DTO) that represents a single, discrete network message within the Hytale protocol. Its sole responsibility is to encapsulate the identifier of a map marker that needs to be removed from a player's user interface.

As an implementation of the Packet interface, it is a fundamental component of the network serialization and deserialization layer. It contains no business logic itself; it is a pure data container. The protocol stack identifies this packet type using its static PACKET_ID field (119) to route raw byte streams from the network to the appropriate static deserialization logic defined within this class.

This class, along with its counterparts, forms the vocabulary of client-server communication for player-specific actions. Its design emphasizes efficiency and strict validation, with methods to compute size, validate structure, and perform serialization directly against a Netty ByteBuf.

### Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **Outbound:** The game logic (e.g., a quest system or player action handler) instantiates it via its constructor when a marker needs to be removed.
    2.  **Inbound:** The network protocol layer instantiates it by invoking the static *deserialize* method when an incoming network buffer with packet ID 119 is processed.
- **Scope:** The object's lifetime is exceptionally short and transient. It exists only long enough to be serialized into a network buffer for transmission, or from the moment of deserialization until it is consumed by a packet handler.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced by the network pipeline or the handling logic. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state consists of a single, mutable, and nullable field: *markerId*. This String holds the unique identifier for the map marker. The object's state is fully defined upon construction or deserialization.

- **Thread Safety:** This class is **not thread-safe**. The public *markerId* field can be accessed and modified without synchronization.

    **WARNING:** This object is designed to be confined to a single thread for its entire lifecycle, typically a Netty I/O thread or a main game loop thread. Sharing instances across threads without external locking will result in race conditions, packet corruption, and undefined behavior. Do not modify its state after it has been submitted to the network layer for serialization.

## API Surface
The public API is designed for interaction with the protocol engine. Application developers will typically only use the constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RemoveMapMarker(String) | constructor | O(1) | Creates a new packet to remove the specified marker. |
| deserialize(ByteBuf, int) | static RemoveMapMarker | O(N) | Constructs a packet by reading from a network buffer. Throws ProtocolException on malformed data. N is the length of the markerId. |
| serialize(ByteBuf) | void | O(N) | Writes the packet's state into a network buffer. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the packet. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a valid packet structure without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (119). |

## Integration Patterns

### Standard Usage
The object should be created, populated, and immediately passed to a network service or channel for transmission. On the receiving end, a handler consumes the deserialized object to perform a game state update.

```java
// Example: Server-side logic sending the packet to a player
// Assume 'playerConnection' is an object managing the network link

String markerToRemove = "quest_objective_alpha";
RemoveMapMarker packet = new RemoveMapMarker(markerToRemove);

playerConnection.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold onto and re-use packet instances. They are lightweight and designed to be created on demand. Re-using an instance can lead to sending stale or incorrect data.
- **Manual Serialization:** Avoid calling *serialize* directly. The network engine is responsible for managing the buffer and the serialization pipeline. Submitting the packet object to the correct service ensures proper handling.
- **Concurrent Access:** Never modify a RemoveMapMarker object from one thread while it is being serialized on another. This will corrupt the network stream.

## Data Pipeline
The RemoveMapMarker packet is a vehicle for data flowing through the network stack.

> **Outbound Flow (Server to Client):**
> Game Event -> Handler creates `new RemoveMapMarker("id")` -> Network Service -> **serialize(ByteBuf)** -> Netty Channel -> TCP Stream

> **Inbound Flow (Client to Server):**
> TCP Stream -> Netty Channel -> Protocol Dispatcher (reads ID 119) -> **deserialize(ByteBuf)** -> Packet Event Bus -> Map System Handler -> Game State Update

