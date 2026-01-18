---
description: Architectural reference for SendWindowAction
---

# SendWindowAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class SendWindowAction implements Packet {
```

## Architecture & Concepts
The SendWindowAction class is a network protocol Data Transfer Object (DTO). It is not a service or manager, but a structured message representing a user's interaction with an in-game user interface window, such as an inventory, chest, or crafting station. This packet is sent from the client to the server to notify it of a specific action performed by the player.

Its design is heavily optimized for high-performance network I/O with the Netty framework. The class adheres to a strict binary layout defined by its static constants, such as FIXED_BLOCK_SIZE and PACKET_ID. This allows the protocol engine to perform serialization, deserialization, and validation directly on raw byte buffers, minimizing object allocation and processing overhead.

The core architectural pattern is the use of a polymorphic payload. The *action* field holds an instance of a subclass of WindowAction, enabling this single packet type to encapsulate a wide variety of UI interactions (e.g., button clicks, item drags, text input) without requiring a unique packet for each.

## Lifecycle & Ownership
- **Creation:** An instance of SendWindowAction is created on the **client** by the UI input handling system. This occurs in direct response to a player action, such as clicking a mouse button within a GUI element. The handler populates the packet with the relevant window ID and the specific WindowAction subclass.

- **Scope:** The object's lifetime is exceptionally brief and transient. On the client, it exists only long enough to be serialized into a ByteBuf by the network layer and queued for transmission. On the server, it is deserialized from a ByteBuf, processed by a single packet handler, and then immediately becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. As it holds no external resources and is not registered with any persistent system, it is cleaned up as soon as it falls out of the scope of the network pipeline or packet handler.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The public fields *id* and *action* can be modified after construction. However, by convention and design, instances should be treated as immutable after their initial creation. They are intended as "write-once, read-many" data carriers.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit external synchronization. It is designed to be created, serialized, and sent on the client's main/UI thread. On the server, it is deserialized and processed within a single Netty worker thread. Concurrent modification will lead to race conditions and unpredictable network data.

## API Surface
The primary contract is with the protocol serialization engine, not general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SendWindowAction(id, action) | constructor | O(1) | Constructs a new packet for transmission. |
| deserialize(buf, offset) | static SendWindowAction | O(N) | Factory method. Deserializes a packet from a raw ByteBuf. N is the size of the action payload. |
| serialize(buf) | void | O(N) | Serializes the packet's state into a ByteBuf for network transmission. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a structural integrity check on the packet data within a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the exact byte size of the packet for buffer pre-allocation. |

## Integration Patterns

### Standard Usage
The packet is created by a client-side system and passed to a network service for dispatch to the server. The server receives and processes it via a registered handler.

```java
// Client-side: A UI event handler creates and sends the packet.
int targetWindowId = 1; // The ID for the currently open inventory
WindowAction clickAction = new ClickSlotAction(5, ClickType.LEFT_CLICK);

SendWindowAction packet = new SendWindowAction(targetWindowId, clickAction);
client.getNetworkManager().sendPacket(packet);

// Server-side: A packet handler processes the incoming data.
public void handleSendWindowAction(PlayerConnection connection, SendWindowAction packet) {
    Player player = connection.getPlayer();
    player.getWindowSystem().handleAction(packet.id, packet.action);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to pool or reuse SendWindowAction objects. They are lightweight and designed for a single-use lifecycle. Reusing instances can lead to corrupted or stale data being sent over the network.

- **Manual Serialization:** Do not manually read from or write to a ByteBuf to simulate this packet. Always use the provided `serialize` and static `deserialize` methods to ensure protocol correctness.

- **Cross-Thread Access:** Never create a packet on one thread and allow another thread to modify it before serialization. All operations on a single instance must be confined to a single thread.

## Data Pipeline
The flow of this data is unidirectional from the client to the server.

> **Client Flow:**
> User Input (e.g., Mouse Click) -> UI Event System -> **new SendWindowAction()** -> Network Manager -> **SendWindowAction.serialize()** -> TCP Socket

> **Server Flow:**
> TCP Socket -> Netty ByteBuf -> Protocol Decoder -> **SendWindowAction.deserialize()** -> Packet Handler -> Game State Update

