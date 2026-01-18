---
description: Architectural reference for BuilderToolSetEntityLight
---

# BuilderToolSetEntityLight

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class BuilderToolSetEntityLight implements Packet {
```

## Architecture & Concepts
The BuilderToolSetEntityLight class is a Data Transfer Object (DTO) that represents a single, discrete network message within the Hytale protocol. It is not a service or a manager; it is a fundamental unit of communication, encapsulating the data required to instruct a remote peer to modify the lighting properties of a specific game entity.

This packet is part of the "Builder Tools" protocol module, indicating its use within the game's creative or editing modes. Its primary architectural role is to serve as a well-defined contract between the client and server for a specific game action.

The design prioritizes network performance and efficiency over flexibility. It employs a fixed-size, binary serialization format managed directly through Netty's ByteBuf. This low-level approach avoids the overhead of reflection-based or text-based formats like JSON. An internal bitmask, `nullBits`, is used to efficiently encode the presence of optional fields, minimizing payload size.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **Outbound (Sending):** Game logic, such as a player interacting with a builder tool, instantiates the packet directly using its constructor: `new BuilderToolSetEntityLight(entityId, light)`. This object is then passed to the network layer for serialization.
    2.  **Inbound (Receiving):** The network protocol layer, upon identifying a packet with ID 422, invokes the static factory method `deserialize(ByteBuf, int)`. This method reads from the raw network buffer and constructs a fully populated object.

- **Scope:** The object's lifetime is extremely short and tied to a single transaction. It exists only to be serialized and sent across the network, or to be deserialized and immediately consumed by a packet handler. It does not persist and is not intended to be stored.

- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as no references to it remain, which is typically immediately after the corresponding network handler has finished processing its data.

## Internal State & Concurrency
- **State:** The class holds a small, mutable state consisting of an entity identifier and an optional ColorLight object. It is a plain data holder with no caching, lazy-loading, or complex internal logic. All fields are public, reflecting its role as a simple DTO.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, such as a Netty I/O worker thread or the main game logic thread. Sharing an instance across threads without external synchronization mechanisms is a severe anti-pattern and will lead to data corruption and race conditions.

## API Surface
The public contract is focused on serialization, deserialization, and metadata for the network protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BuilderToolSetEntityLight | O(1) | Factory method. Constructs an instance by reading from a network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a network buffer for transmission. |
| getId() | int | O(1) | Returns the unique, static network identifier for this packet type (422). |
| computeSize() | int | O(1) | Returns the fixed size in bytes (9) that this packet occupies on the wire. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-deserialization check to ensure the buffer is large enough to contain the packet. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol dispatcher and game systems that handle builder tool events. A system creates the packet, passes it to a network service for sending, or receives it from a network handler for processing.

```java
// Example: Sending the packet from game logic
int targetEntityId = 123;
ColorLight newLight = new ColorLight(255, 200, 150, 15);

// 1. Instantiate the packet with game data
BuilderToolSetEntityLight packet = new BuilderToolSetEntityLight(targetEntityId, newLight);

// 2. Pass to the network connection manager to serialize and send
// networkManager.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify a packet object after it has been passed to the network layer for sending. Do not hold references to received packets after they have been processed. They are cheap to create and should be treated as immutable messages once populated.
- **Manual Serialization/Deserialization:** Do not bypass the provided `serialize` and `deserialize` methods to read or write from a ByteBuf manually. The internal layout, including the `nullBits` field, is an implementation detail.
- **Cross-Thread Modification:** Never create a packet on one thread and modify its fields on another. All operations on a single instance must be confined to a single thread or protected by external locks.

## Data Pipeline
The BuilderToolSetEntityLight packet is a data record that flows through the network stack.

> **Outbound Flow:**
> Game Logic -> `new BuilderToolSetEntityLight()` -> Network Encoder -> **serialize()** -> TCP/UDP Socket

> **Inbound Flow:**
> TCP/UDP Socket -> Network Decoder -> Packet Dispatcher -> **deserialize()** -> Game Event Handler

