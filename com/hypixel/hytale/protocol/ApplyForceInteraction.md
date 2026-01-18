---
description: Architectural reference for ApplyForceInteraction
---

# ApplyForceInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ApplyForceInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The **ApplyForceInteraction** class is a Data Transfer Object (DTO) that represents a specific command within Hytale's binary network protocol. It is not a service or a manager; its sole purpose is to model the data required to apply one or more physical forces to a game entity over a duration. This class acts as a structured container for data that is serialized for network transmission and deserialized upon receipt.

Architecturally, it sits at the boundary between the low-level network transport layer (managed by Netty) and the high-level game simulation logic. Its design is heavily optimized for network performance, employing a custom binary layout to minimize packet size.

This layout consists of two primary sections:
1.  **Fixed-Size Block:** A contiguous block of 80 bytes containing primitive types like floats, integers, and booleans. This allows for extremely fast, direct-offset reads and writes.
2.  **Variable-Size Block:** A subsequent data region that contains complex, variable-length data such as arrays and nested objects. The fixed-size block contains integer offsets pointing to the start of each variable field within this block.

A single byte, referred to as `nullBits`, acts as a bitfield at the start of the payload. Each bit corresponds to a nullable, variable-sized field, indicating whether its data is present in the packet. This avoids wasting space transmitting null markers or empty data structures.

## Lifecycle & Ownership
-   **Creation:** An **ApplyForceInteraction** instance is created under two circumstances:
    1.  **Inbound:** The static factory method `deserialize` is invoked by a network channel handler when an incoming network packet of this type is decoded from a Netty ByteBuf.
    2.  **Outbound:** Game logic (e.g., a combat or physics system) instantiates the class directly using its constructor, populates its public fields, and submits it to the network layer for serialization.
-   **Scope:** Instances are ephemeral and have a very short lifecycle. They exist only for the time it takes to be serialized into a buffer for sending, or for the time it takes for a game logic handler to process the data from a deserialized instance. They do not persist between game ticks.
-   **Destruction:** The object is managed by the Java Garbage Collector. Once it has been processed by the relevant game system, it falls out of scope and is cleaned up. There are no manual resource management requirements.

## Internal State & Concurrency
-   **State:** The object's state is entirely defined by its public fields. It is a fully mutable data container with no internal logic for state management or validation beyond the initial deserialization checks. It holds no references to persistent game state.
-   **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it susceptible to race conditions if accessed concurrently. The protocol framework guarantees that deserialization and subsequent processing occur within a single, designated thread (e.g., a Netty event loop thread or the main game thread).

    **WARNING:** Never share an instance of **ApplyForceInteraction** between threads without explicit synchronization or safe handoff mechanisms like a thread-safe queue.

## API Surface
The primary contract of this class is its data structure and the static methods for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ApplyForceInteraction | O(N) | **[Factory]** Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf using the custom binary format. Returns bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the buffer to ensure offsets and lengths are valid before attempting a full deserialization. Critical for security. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object within a buffer without creating an instance. |
| clone() | ApplyForceInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol handlers. A handler decodes the message and dispatches it to the appropriate game system for processing.

```java
// In a network channel handler or dispatcher
ByteBuf incomingPacket = ...;
int offset = ...;

// Validate before processing to prevent protocol errors
ValidationResult result = ApplyForceInteraction.validateStructure(incomingPacket, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid ApplyForceInteraction: " + result.error());
}

// Deserialize and pass to game logic
ApplyForceInteraction interaction = ApplyForceInteraction.deserialize(incomingPacket, offset);
gameWorld.getPhysicsSystem().processInteraction(entityId, interaction);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not hold references to **ApplyForceInteraction** objects past the current game tick. They are transient data packets, not components of the game state.
-   **Concurrent Modification:** Do not read from or write to an instance from multiple threads. If a network thread deserializes the object, it must be safely published to the main game thread before use.
-   **Manual Serialization:** Do not attempt to manually write the fields to a buffer. The binary layout is complex, involving null bitfields and relative offsets. Always use the provided `serialize` method to ensure correctness.

## Data Pipeline
The class serves as a data payload in a client-server communication pipeline.

> **Outbound Flow:**
> Game Event -> Game Logic creates **ApplyForceInteraction** -> `serialize(buffer)` -> Network Encoder -> TCP Socket

> **Inbound Flow:**
> TCP Socket -> Netty ByteBuf -> Network Decoder -> `validateStructure()` -> `deserialize()` -> **ApplyForceInteraction** instance -> Game Logic Handler -> Physics Engine Update

