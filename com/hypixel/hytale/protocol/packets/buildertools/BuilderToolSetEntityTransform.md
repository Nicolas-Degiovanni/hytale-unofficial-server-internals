---
description: Architectural reference for BuilderToolSetEntityTransform
---

# BuilderToolSetEntityTransform

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BuilderToolSetEntityTransform implements Packet {
```

## Architecture & Concepts
The BuilderToolSetEntityTransform class is a network **Packet**, a specialized Data Transfer Object (DTO) designed for high-throughput communication between the client and server. Its sole purpose is to encapsulate the data required to modify an entity's physical transformation—its position, rotation, and scale—as part of the in-game builder tools feature set.

This class is a fundamental component of the Hytale Protocol Layer. It is not a service or a manager; it is a pure data container. Its design prioritizes performance and low-level network efficiency over encapsulation. The key architectural decision is its **fixed 54-byte size**, which simplifies buffer allocation and parsing logic within the Netty-based network pipeline. This avoids the overhead of dynamic size calculation or length-prefixing for this specific message type, making it ideal for frequent, low-latency updates like moving an object in a world editor.

The serialization and deserialization logic operates directly on Netty's ByteBuf, minimizing memory copies and object allocation overhead during network I/O operations.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderToolSetEntityTransform is created under two distinct circumstances:
    1.  **On the sending endpoint** (e.g., a game client), it is instantiated directly using its constructor (`new BuilderToolSetEntityTransform(...)`) when a user performs an action that modifies an entity's transform.
    2.  **On the receiving endpoint** (e.g., a game server), it is never instantiated with `new`. Instead, it is created by the protocol layer's packet dispatcher, which invokes the static `deserialize` factory method on a raw network ByteBuf.

- **Scope:** This object is extremely **short-lived and ephemeral**. It is designed to exist only for the duration of a single network event processing cycle. Once the packet is serialized and sent, or received and processed, it should be considered stale and is immediately eligible for garbage collection.

- **Destruction:** Destruction is managed by the Java Garbage Collector. There are no manual cleanup or `close` methods. Because of its transient nature, it is typically garbage collected very quickly after its data has been applied to the game world state.

## Internal State & Concurrency
- **State:** The class is a **mutable** data holder. Its fields are public and can be modified directly after instantiation. This design choice sacrifices encapsulation for raw performance, a common trade-off in low-level networking code. It contains no internal caches or complex state beyond the data it transports.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, serialized, deserialized, and processed within a single, well-defined thread, such as a Netty event loop thread or the main game thread. Accessing or modifying an instance from multiple threads without external synchronization will result in data corruption and undefined behavior.

## API Surface
The public contract is focused on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BuilderToolSetEntityTransform(int, ModelTransform) | constructor | O(1) | Creates a new packet instance for sending. |
| serialize(ByteBuf) | void | O(1) | Writes the packet's fixed 54-byte representation into the buffer. |
| deserialize(ByteBuf, int) | static BuilderToolSetEntityTransform | O(1) | Reads 54 bytes from the buffer to construct a new packet instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer is large enough to contain the packet. |
| computeSize() | int | O(1) | Returns the constant size of the packet, which is always 54. |

## Integration Patterns

### Standard Usage
This packet is typically handled within a protocol dispatcher or packet handler, where a raw buffer is decoded into a specific packet type based on its ID.

```java
// Example: In a server-side packet handler
// The network layer has already read the packet ID and determined it is 402.

public void handlePacket(ByteBuf incomingBuffer, int offset) {
    ValidationResult result = BuilderToolSetEntityTransform.validateStructure(incomingBuffer, offset);
    if (!result.isOk()) {
        // Handle error, disconnect client, or log a warning
        return;
    }

    BuilderToolSetEntityTransform packet = BuilderToolSetEntityTransform.deserialize(incomingBuffer, offset);
    
    // Apply the transform to the world state within the main game thread
    World world = gameContext.getWorld();
    Entity targetEntity = world.getEntityById(packet.entityId);
    if (targetEntity != null) {
        targetEntity.applyTransform(packet.modelTransform);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Storage:** Do not store instances of this packet in long-lived collections or as member variables in services. It represents a single, point-in-time event, not persistent application state.

- **Reusing Instances:** Do not attempt to modify and re-serialize the same packet instance. For clarity and safety, create a new instance for each distinct network message.

- **Cross-Thread Access:** Never pass a packet instance from a network thread to a worker thread without creating a defensive copy or using a thread-safe data structure. Direct sharing will lead to severe concurrency issues.

## Data Pipeline
The flow of data for this packet is linear and unidirectional for a single event.

> Flow:
> Client User Input -> Game Logic -> **BuilderToolSetEntityTransform.serialize()** -> Network Channel -> Server Network Channel -> **BuilderToolSetEntityTransform.deserialize()** -> Server Game Logic -> World State Update

