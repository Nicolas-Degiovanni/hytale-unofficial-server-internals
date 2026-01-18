---
description: Architectural reference for BuilderToolSetEntityPickupEnabled
---

# BuilderToolSetEntityPickupEnabled

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolSetEntityPickupEnabled implements Packet {
```

## Architecture & Concepts
The BuilderToolSetEntityPickupEnabled class is a network Packet, a specialized Data Transfer Object (DTO) designed for client-server communication. It represents a specific, atomic command within the Builder Tools subsystem. Its sole purpose is to transmit a state change—enabling or disabling the "pickup" behavior for a targeted game entity—from a client with appropriate permissions to the server.

This packet is part of a command-based protocol, where client actions are serialized into discrete, well-defined messages. It is not a service or a manager; it is inert data. The protocol machinery on both the client and server is responsible for its serialization, transmission, and deserialization. Its fixed-size structure, defined by constants like FIXED_BLOCK_SIZE and PACKET_ID, is critical for high-performance network processing, allowing the server to quickly identify, validate, and decode the incoming byte stream without complex parsing logic.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client when a player, using a builder tool, modifies an entity's pickup property. The client-side game logic instantiates this object, populating it with the target entity's unique ID and the new boolean state.
- **Scope:** The lifecycle of this object is exceptionally brief and confined to the network transaction. It exists only for the moments it takes to be serialized by the sender, transmitted over the network, and deserialized by the receiver.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by the server-side packet handler. No long-term references should ever be held to a packet instance.

## Internal State & Concurrency
- **State:** The object holds a small, mutable state consisting of an integer (entityId) and a boolean (enabled). While technically mutable, instances of this packet should be treated as immutable after creation. Its state is a snapshot of a command at a single point in time.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data holder with no internal locking. This is by design. Network packets are processed sequentially by a dedicated network thread or handed off to a main game loop thread. They are never intended to be accessed concurrently by multiple threads.

## API Surface
The public API is minimal, focusing entirely on the serialization and deserialization contract required by the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (421). |
| serialize(ByteBuf) | void | O(1) | Writes the entityId and enabled flag into the provided Netty byte buffer. |
| deserialize(ByteBuf, int) | BuilderToolSetEntityPickupEnabled | O(1) | Static factory method. Reads 5 bytes from the buffer to construct a new packet instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload in bytes (5). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough data to read the packet. |

## Integration Patterns

### Standard Usage
This packet is never used directly by feature developers. It is created and dispatched into the network layer, which handles its lifecycle. The server-side logic receives the deserialized object from the network layer.

```java
// Client-side: Sending the command
// This logic resides deep within a builder tool or entity interaction system.
int targetEntityId = 12345;
boolean canBePickedUp = false;
BuilderToolSetEntityPickupEnabled packet = new BuilderToolSetEntityPickupEnabled(targetEntityId, canBePickedUp);
clientConnection.sendPacket(packet);

// Server-side: A packet handler receives the object
// This is a conceptual handler; actual implementation may vary.
public void handleSetEntityPickup(BuilderToolSetEntityPickupEnabled packet) {
    World world = server.getWorld();
    Entity target = world.findEntityById(packet.entityId);
    if (target != null) {
        target.setPickupEnabled(packet.enabled);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify a packet's fields after it has been passed to the network system. This can lead to data corruption or desynchronization.
- **Manual Serialization:** Never call serialize or deserialize directly. These methods are strictly for use by the core protocol engine, which manages buffer allocation and packet framing.
- **Object Caching:** Do not cache or reuse packet objects. They are extremely lightweight and should be instantiated for each new command to ensure thread safety and state integrity.

## Data Pipeline
The flow of this command is unidirectional from client to server.

> Flow:
> Client User Input -> Builder Tool System -> **new BuilderToolSetEntityPickupEnabled()** -> Network Serialization Engine -> TCP Stream -> Server Network Listener -> Packet Deserialization Engine -> Packet Dispatcher -> Game Logic Handler -> World State Mutation

