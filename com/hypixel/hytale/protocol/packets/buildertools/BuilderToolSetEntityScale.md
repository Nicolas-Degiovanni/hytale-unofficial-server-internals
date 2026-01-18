---
description: Architectural reference for BuilderToolSetEntityScale
---

# BuilderToolSetEntityScale

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolSetEntityScale implements Packet {
```

## Architecture & Concepts
The BuilderToolSetEntityScale class is a **Packet**, a specialized Data Transfer Object (DTO) designed for network communication. It does not contain any game logic. Its sole purpose is to represent a single, atomic command within the Hytale network protocol: changing the scale of a game entity.

This packet is part of the "Builder Tools" feature set, used in creative or world-editing game modes. As a protocol entity, it is defined by a strict, fixed-size binary layout, optimized for high-performance serialization and deserialization. The static constants within the class, such as PACKET_ID, FIXED_BLOCK_SIZE, and MAX_SIZE, define this binary contract, ensuring that both client and server can interpret the network stream correctly.

It acts as a data record that flows from a sender (e.g., a client executing a command) to a receiver (e.g., a server applying the world state change).

## Lifecycle & Ownership
- **Creation:**
    - **On the sending endpoint:** An instance is created directly when a user action triggers the command. For example, `new BuilderToolSetEntityScale(entityId, 1.5f);`. This object is then immediately passed to the network layer for serialization.
    - **On the receiving endpoint:** An instance is created by the protocol layer's packet factory. The static method `deserialize` is invoked by a central dispatcher after it reads the packet ID (420) from the incoming network buffer.

- **Scope:** The lifecycle of this object is extremely brief and ephemeral. It exists only for the duration of a single network operation. It is created, serialized (or deserialized), processed by a single handler, and then immediately becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures. It is expected to be dereferenced and collected shortly after its data has been consumed by the game logic.

## Internal State & Concurrency
- **State:** The class holds a simple, mutable state consisting of an `entityId` and a `scale`. While the fields are public and mutable, instances are treated as immutable value objects in practice. They are not designed to be modified after initial population.

- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives. It is designed to be created, populated, and processed within a single thread, typically a Netty I/O thread or the main game logic thread. Sharing an instance of this packet across threads without explicit external synchronization is a critical error and will lead to race conditions.

## API Surface
The public API is focused on protocol-level operations for encoding and decoding the packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BuilderToolSetEntityScale | O(1) | Constructs a new packet by reading a fixed 8-byte block from the buffer at the given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal state (entityId, scale) as a fixed 8-byte block into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet, which is 8 bytes. |
| getId() | int | O(1) | Returns the unique network identifier for this packet type, which is 420. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
This packet is used by higher-level systems to dispatch commands over the network. It should never be managed directly by gameplay feature developers.

**Sending a Packet:**
```java
// In a system that handles builder tool actions
int targetEntity = 123;
float newScale = 2.0f;

BuilderToolSetEntityScale packet = new BuilderToolSetEntityScale(targetEntity, newScale);
networkConnection.sendPacket(packet);
```

**Receiving a Packet:**
A central packet dispatcher, not shown here, is responsible for reading the packet ID from the wire and calling the static `deserialize` method. The resulting object is then routed to a registered handler.

```java
// In a registered packet handler
public void handleSetEntityScale(BuilderToolSetEntityScale packet) {
    GameWorld world = context.getWorld();
    Entity entity = world.getEntityById(packet.entityId);
    if (entity != null) {
        entity.setScale(packet.scale);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and modify a packet instance to send multiple commands. They are lightweight objects, and creating a new instance for each command prevents state corruption.
- **Cross-Thread Modification:** Never deserialize a packet on a network thread and pass the packet object itself to a game logic thread for processing. Instead, extract the primitive data (`entityId`, `scale`) and pass those values to a thread-safe work queue.
- **Manual Serialization:** Do not bypass the `serialize` and `deserialize` methods to read or write the fields directly to a ByteBuf. This would break the protocol contract and lead to deserialization errors.

## Data Pipeline
The BuilderToolSetEntityScale packet is a data container that moves through a well-defined serialization and deserialization pipeline.

> **Outbound Flow (e.g., Client to Server):**
> User Input -> Builder Tool Logic -> `new BuilderToolSetEntityScale()` -> Network System -> **serialize()** -> Netty I/O Thread -> TCP Socket

> **Inbound Flow (e.g., Server from Client):**
> TCP Socket -> Netty I/O Thread -> Protocol Decoder -> Packet Dispatcher (reads ID 420) -> **deserialize()** -> **BuilderToolSetEntityScale instance** -> Packet Handler -> Game World State Update

