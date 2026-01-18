---
description: Architectural reference for UpdateWorldMapVisible
---

# UpdateWorldMapVisible

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class UpdateWorldMapVisible implements Packet {
```

## Architecture & Concepts
UpdateWorldMapVisible is a simple Data Transfer Object (DTO) that represents a single, specific command within the Hytale network protocol. Its sole purpose is to communicate a change in the visibility state of the player's world map from the server to the client.

As an implementation of the Packet interface, this class is a fundamental building block of the network layer. It is not a service or a manager; it is a message. The class structure is optimized for network performance, featuring a fixed, minimal size (1 byte) and static methods for serialization and deserialization that operate directly on Netty ByteBufs. This design avoids unnecessary object allocations and computational overhead during network I/O processing.

This packet is part of the "worldmap" sub-protocol, which governs all interactions related to the in-game map system, such as marker updates, fog of war changes, and visibility toggles.

### Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by game logic when an event requires the client's map UI to be shown or hidden. For example, entering a specific zone or interacting with a map table.
    - **Client-Side:** Instantiated exclusively by the network protocol layer via the static *deserialize* method when a corresponding packet (ID 243) is read from the incoming network buffer.
- **Scope:** The lifetime of an UpdateWorldMapVisible instance is extremely brief. It is created, processed, and then immediately becomes eligible for garbage collection. It is a fire-and-forget message.
- **Destruction:** There is no explicit destruction logic. The Java Garbage Collector reclaims the memory once the packet handler has finished processing it.

## Internal State & Concurrency
- **State:** The class holds a single mutable boolean field: *visible*. This represents the desired state of the world map on the client. The object's state is set once upon creation and is not intended to be modified thereafter.
- **Thread Safety:** This class is **not thread-safe**. Instances are designed to be confined to a single thread, typically a Netty I/O worker thread. All processing, from deserialization to handling, must occur sequentially within the context of the network event loop to prevent data corruption and race conditions. The static methods are stateless and inherently safe to call from any context, though they are only intended for use by the packet dispatcher.

## API Surface
The public API is minimal, focusing entirely on the contract required by the Packet interface and the network processing pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (243). |
| serialize(ByteBuf) | void | O(1) | Writes the boolean *visible* state as a single byte to the provided buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload, which is always 1 byte. |
| deserialize(ByteBuf, int) | UpdateWorldMapVisible | O(1) | **Static Factory.** Reads 1 byte from the buffer and constructs a new packet instance. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly. Instead, logic is implemented in a packet handler that is registered to receive this specific packet type. The network layer dispatches the deserialized object to the handler.

```java
// Example of a client-side packet handler
public class WorldMapPacketHandler implements PacketHandler<UpdateWorldMapVisible> {

    private final WorldMapUI mapUI;

    @Override
    public void handle(UpdateWorldMapVisible packet) {
        // The packet's data is used to update the state of a UI component.
        // This logic is executed on the main client thread after being
        // dispatched from the network thread.
        boolean isVisible = packet.visible;
        this.mapUI.setVisibility(isVisible);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Do not use `new UpdateWorldMapVisible()` on the client. The client's role is to receive and react to this packet, not create it.
- **State Reuse:** Do not hold references to packet instances after they have been handled. They are transient and should be considered invalid after the handler completes.
- **Manual Serialization/Deserialization:** Avoid calling *serialize* or *deserialize* outside the core network engine. The protocol pipeline is designed to manage the entire lifecycle. Manual calls can lead to buffer corruption and de-synchronization.

## Data Pipeline
The flow of data for this packet is unidirectional from server to client.

> **Server Flow:**
> Game Event -> Game Logic -> `new UpdateWorldMapVisible(state)` -> Network Channel -> **serialize()** -> TCP Stream

> **Client Flow:**
> TCP Stream -> Netty ByteBuf -> Packet Dispatcher -> **deserialize()** -> `UpdateWorldMapVisible` Instance -> Packet Handler -> UI State Update

