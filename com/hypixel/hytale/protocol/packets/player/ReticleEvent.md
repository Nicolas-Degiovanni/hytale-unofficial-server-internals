---
description: Architectural reference for ReticleEvent
---

# ReticleEvent

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ReticleEvent implements Packet {
```

## Architecture & Concepts
The ReticleEvent class is a network packet Data Transfer Object (DTO) that represents a discrete, client-initiated event related to the player's user interface reticle, commonly known as the crosshair. It serves as a lightweight, fixed-size message to communicate simple state changes or actions from the game client to the server.

As an implementation of the Packet interface, this class is a fundamental component of the client-server communication protocol. Its structure is optimized for high-performance network processing. The static final fields such as PACKET_ID, FIXED_BLOCK_SIZE, and IS_COMPRESSED are not for direct developer use; they are metadata consumed by the core protocol engine to route, deserialize, and manage incoming byte streams from the network without expensive reflection or type-checking.

The single data field, eventIndex, is an integer that likely maps to a server-side enumeration or constant, representing specific reticle actions like "target acquired," "target lost," or "weapon charge level changed." This design avoids sending expensive string data over the network.

## Lifecycle & Ownership
-   **Creation:** An instance of ReticleEvent is created on the client whenever a gameplay event involving the reticle needs to be communicated to the server. On the server, a new instance is created by the protocol's deserialization layer when a byte stream with PACKET_ID 113 is received.
-   **Scope:** The object's lifetime is extremely brief and transactional. It exists only long enough to be serialized into a network buffer on the client, or to be deserialized and processed by a packet handler on the server.
-   **Destruction:** The object holds no persistent state or native resources. It is managed entirely by the Java Garbage Collector and becomes eligible for collection immediately after its data has been processed by the relevant game system.

## Internal State & Concurrency
-   **State:** The internal state consists of a single, mutable public integer field: eventIndex. The class is a simple data container with no internal logic or caching. Its state is entirely defined by the value of this field at the time of serialization or after deserialization.
-   **Thread Safety:** This class is **not thread-safe**. It is a plain data structure with no internal synchronization mechanisms. It is designed to be created, populated, and processed within a single-threaded context, such as a Netty I/O thread or a main game loop tick.

**WARNING:** Modifying a ReticleEvent instance from multiple threads without external locking will result in undefined behavior and data corruption. It should be treated as an immutable object after it has been submitted to the network pipeline.

## API Surface
The primary contract is defined by the Packet interface and the static serialization methods, which are invoked by the protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the eventIndex into the provided network buffer. Called by the protocol engine. |
| deserialize(ByteBuf, int) | static ReticleEvent | O(1) | Decodes a network buffer into a new ReticleEvent instance. Called by the protocol engine. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload, which is 4 bytes. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a low-level check to ensure a buffer has enough readable bytes for this packet. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated by a client-side system, populated, and passed to the network layer for transmission. Server-side code receives the deserialized object in a registered packet handler.

```java
// Client-side: Firing a reticle event
// Assume some UI system detects a "target acquired" event, mapped to index 2.
ReticleEvent event = new ReticleEvent(2);
clientConnection.sendPacket(event);

// Server-side: A packet handler receives the object
public void handleReticleEvent(PlayerConnection connection, ReticleEvent event) {
    // Process the event based on its index
    if (event.eventIndex == 2) {
        connection.getPlayer().getCombatSystem().setTargetAcquired(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Object Reuse:** Do not modify and resend the same ReticleEvent instance. This can cause severe bugs if the object is still referenced by the network pipeline. Always create a new object for each distinct event.
-   **Direct Serialization:** Do not call serialize or deserialize directly. These are low-level methods intended for the core protocol engine. Interacting with them directly bypasses critical pipeline handlers for compression, encryption, and batching.
-   **Server-to-Client Transmission:** This is a client-to-server packet. Sending it from the server to the client is a protocol violation and will likely result in the client disconnecting.

## Data Pipeline
The ReticleEvent packet follows a simple, unidirectional flow from the client to the server.

> Flow:
> Client UI Input -> **ReticleEvent (Instantiation)** -> Network Pipeline (Serialization) -> TCP/UDP Transmission -> Server Network Pipeline (Deserialization) -> **ReticleEvent (Re-creation)** -> Packet Handler -> Server Game Logic

---

