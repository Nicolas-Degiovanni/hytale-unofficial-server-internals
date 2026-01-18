---
description: Architectural reference for BuilderToolLineAction
---

# BuilderToolLineAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BuilderToolLineAction implements Packet {
```

## Architecture & Concepts
The BuilderToolLineAction class is a network packet definition within the Hytale Protocol Layer. It serves as a specialized Data Transfer Object designed to communicate a single, discrete in-game event: a user drawing a line with a builder tool.

This class is not a service or a manager; it is a pure data container. Its primary architectural role is to provide a language-neutral, fixed-size contract for serializing a line-drawing command between a client and a server. The design prioritizes performance and predictability over encapsulation. The use of public fields and a fixed block size of 24 bytes allows the network layer to perform highly efficient, zero-overhead read and write operations directly on Netty ByteBufs without the need for reflection or complex parsing logic.

This packet is an example of a *Command* pattern implementation over the network, where the packet itself represents a command to be executed by the receiving system.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (e.g., Client):** Instantiated directly via its constructor (`new BuilderToolLineAction(...)`) by game logic when a player finalizes a line-drawing action. The object is then passed to the network subsystem for serialization.
    - **Receiving Peer (e.g., Server):** Instantiated by the protocol's packet dispatcher, which calls the static `deserialize` factory method. This occurs when an incoming data stream contains a packet with ID 414.

- **Scope:** Extremely short-lived and transient. An instance of BuilderToolLineAction exists only for the duration of a single network event processing cycle. It is created, passed to a handler, and immediately becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. Once the network handler or game logic has processed the data, no strong references remain, and the memory is reclaimed.

## Internal State & Concurrency
- **State:** The state is mutable and consists of six 32-bit integer coordinates representing the start and end points of a line. All state is exposed via public fields. This is a deliberate design choice for high-performance network DTOs, avoiding the overhead of method calls for data access. The object is stateless in the sense that it contains no caches, connections, or references to other systems.

- **Thread Safety:** **This class is not thread-safe.** The public mutable fields make it fundamentally unsafe for concurrent access. It is designed to be created on one thread, serialized, and then deserialized and processed on a single network or game-logic thread.

    **WARNING:** Sharing an instance of this class across multiple threads without external synchronization mechanisms (e.g., locks or thread-safe queues) will lead to race conditions and data corruption. It should be treated as thread-local during processing.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Serializes the six integer fields into the provided ByteBuf using little-endian byte order. |
| deserialize(ByteBuf, int) | BuilderToolLineAction | O(1) | **Static Factory.** Reads 24 bytes from a buffer and constructs a new BuilderToolLineAction instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload, which is always 24 bytes. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Method.** Performs a pre-check to ensure the buffer contains at least 24 readable bytes before attempting deserialization. |
| getId() | int | O(1) | Returns the unique network identifier for this packet type, which is always 414. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the core network protocol handlers. A handler identifies the packet ID from the stream, validates the buffer size, and uses the static `deserialize` method to create the object. The resulting instance is then passed to a higher-level game system for processing.

```java
// Example server-side packet handler
void handlePacket(ByteBuf buffer) {
    // Assume packet ID 414 was already read
    ValidationResult result = BuilderToolLineAction.validateStructure(buffer, buffer.readerIndex());
    if (result.isError()) {
        // Handle error, disconnect client
        return;
    }

    BuilderToolLineAction action = BuilderToolLineAction.deserialize(buffer, buffer.readerIndex());
    
    // Pass the immutable data to the game world thread
    gameWorld.getBuilderToolSystem().applyLineAction(action);
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification on Receiver:** Do not modify the fields of a deserialized BuilderToolLineAction object. Treat it as an immutable record of a past event. If modification is needed, create a new object.
- **Cross-Thread Sharing:** Never pass a reference to a single BuilderToolLineAction instance to multiple worker threads for concurrent processing. This will cause unpredictable behavior due to the public mutable fields.
- **Manual Deserialization:** Do not manually read integers from the ByteBuf. Always use the provided static `deserialize` method to ensure correctness and forward compatibility.

## Data Pipeline
The BuilderToolLineAction packet is a simple, single-stage data carrier. Its flow is linear and unidirectional within a single transaction.

> **Client Flow:**
> Player Input -> Builder Tool Logic -> `new BuilderToolLineAction()` -> Network Encoder -> **serialize()** -> TCP Stream

> **Server Flow:**
> TCP Stream -> Network Decoder -> Packet Dispatcher (ID 414) -> **deserialize()** -> Game Logic Handler -> World State Update

