---
description: Architectural reference for BuilderToolSetNPCDebug
---

# BuilderToolSetNPCDebug

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolSetNPCDebug implements Packet {
```

## Architecture & Concepts
The BuilderToolSetNPCDebug class is a pure Data Transfer Object (DTO) that represents a specific, fixed-size network message within the Hytale protocol. It is not a service or manager; its sole responsibility is to encapsulate the data required to toggle a debug visualization for a specific Non-Player Character (NPC) via the in-game builder tools.

This class acts as a strict data contract between the client and server, identified uniquely by the static PACKET_ID 423. Its core architectural function is to provide a type-safe, Java-native representation of a low-level binary network command. The serialization and deserialization logic, which directly manipulates Netty ByteBufs, defines the precise 5-byte layout of this command on the wire.

The presence of static factory methods like *deserialize* and *validateStructure* is a key design pattern in the protocol layer. It allows the network dispatcher to validate and construct packets from a raw byte stream without needing a pre-existing object instance, which is essential for high-performance network I/O processing.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Peer:** Instantiated directly via its constructor, for example `new BuilderToolSetNPCDebug(entityId, true)`, typically in response to a user input event.
    - **Receiving Peer:** Instantiated by the protocol's packet dispatcher, which invokes the static `deserialize` factory method upon identifying PACKET_ID 423 in the incoming byte stream.

- **Scope:** Extremely short-lived and ephemeral. An instance of this class exists only for the brief moment between its creation and its serialization, or between its deserialization and its processing by a network event handler. It is a "fire-and-forget" message container.

- **Destruction:** The object is immediately eligible for garbage collection after its data has been consumed by a handler or written to a network buffer. It holds no resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data holder with public fields. It contains no internal caches, lazy-loaded data, or references to external systems. Its state is comprised entirely of the `entityId` and `enabled` fields.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, such as a Netty I/O worker or a main game thread. Accessing or modifying an instance from multiple threads without external locking mechanisms will result in undefined behavior and data corruption.

## API Surface
The public contract is focused on protocol-level encoding and decoding operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (423) for this packet type. |
| serialize(ByteBuf buf) | void | O(1) | Encodes the object's state into the provided buffer according to the protocol's binary specification. |
| deserialize(ByteBuf buf, int offset) | static BuilderToolSetNPCDebug | O(1) | **Factory Method.** Decodes a binary buffer into a new object instance. Throws if buffer is malformed. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a low-level size check on a raw buffer before attempting deserialization. |

## Integration Patterns

### Standard Usage
This packet is created and dispatched to the network layer. It should never be held onto or modified after being sent.

```java
// Example: Sending the packet from a client-side input handler
int targetNpcId = getTargetedEntityId();
boolean shouldEnableDebug = true;

// Create a new packet for each distinct command
BuilderToolSetNPCDebug packet = new BuilderToolSetNPCDebug(targetNpcId, shouldEnableDebug);
networkManager.sendPacket(packet);
```

On the receiving end, a registered handler consumes the packet data.

```java
// Example: A server-side packet handler
public void handleSetNpcDebug(BuilderToolSetNPCDebug packet) {
    World world = server.getWorld();
    Entity npc = world.getEntityById(packet.entityId);
    if (npc != null) {
        npc.setDebugRender(packet.enabled);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not modify a packet object after it has been sent and attempt to send it again. This is not safe, as the network layer may be processing the object on a different thread. Always create a new instance for each command.
- **Manual Serialization:** Do not bypass the `serialize` and `deserialize` methods to read or write fields directly to a ByteBuf. This will break protocol compatibility if the binary layout ever changes.
- **State Management:** Do not store instances of this packet in long-lived collections or use them to hold application state. They are messages, not state containers.

## Data Pipeline
The flow of this data object is linear, moving from a game event, across the network, and into a game state update on the remote peer.

> Flow (Client to Server):
> User Input -> Game Logic creates **BuilderToolSetNPCDebug** -> Protocol Encoder calls `serialize` -> TCP/IP Stack -> Server Network Layer -> Protocol Decoder calls `deserialize` -> Packet Handler -> World State Update

