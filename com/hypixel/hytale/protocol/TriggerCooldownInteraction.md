---
description: Architectural reference for TriggerCooldownInteraction
---

# TriggerCooldownInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class TriggerCooldownInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The TriggerCooldownInteraction class is a passive data structure that defines a specific type of game interaction packet. It is not an active, managed service but rather a schematic for serializing and deserializing game mechanics data sent from the server to the client.

As a subclass of SimpleInteraction, it represents one of many possible player or world interactions. Its primary architectural role is to act as the on-the-wire contract for interactions that involve a cooldown period. The class design is heavily optimized for network performance, employing a custom binary format.

The serialization format consists of two main parts:
1.  **Fixed Block:** A 43-byte header containing primitive types and offsets. This includes a `nullBits` byte which acts as a bitmask to efficiently signal the presence of optional, variable-sized data fields.
2.  **Variable Block:** A subsequent data region where complex types like maps (settings), arrays (tags), and nested objects (effects, rules, cooldown) are stored. The fixed block contains integer offsets pointing to the start of each of these structures within the variable block.

This design allows for extremely fast parsing and size calculation, as the location and presence of all data can be determined by reading the initial fixed-size header.

## Lifecycle & Ownership

-   **Creation:** An instance of TriggerCooldownInteraction is created under two distinct circumstances:
    1.  **Server-Side:** Instantiated by game logic systems on the server, populated with specific interaction rules, and then passed to the network layer for serialization.
    2.  **Client-Side:** Instantiated exclusively by the static `deserialize` method when a corresponding packet is read from the network buffer by a Netty channel handler.
-   **Scope:** The object is ephemeral and has a very short lifecycle. Its scope is typically confined to the processing of a single network packet. Once deserialized, its data is consumed by a client-side system, and the object is then eligible for garbage collection.
-   **Destruction:** There is no explicit destruction method. The Java Garbage Collector manages its memory once all references to the instance are dropped.

## Internal State & Concurrency

-   **State:** The object's state is fully mutable. All fields are public and are directly manipulated during the deserialization process. While mutable by design for performance, a deserialized instance should be treated as an **immutable value object** by consuming systems. Modifying its state after deserialization can lead to unpredictable behavior in the game client.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, serialized, or deserialized and processed within a single thread, such as a Netty I/O thread or the main game loop thread.

**WARNING:** Concurrent access to an instance of TriggerCooldownInteraction from multiple threads will result in data corruption and race conditions. Do not share instances across threads.

## API Surface

The primary API is static and centered around serialization and validation from a raw ByteBuf.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | TriggerCooldownInteraction | O(N) | **Primary Constructor.** Deserializes binary data from a buffer into a new object instance. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | **Primary Egress Point.** Serializes the object's state into the provided buffer. Throws ProtocolException if internal collections exceed protocol limits. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only structural validation of the data in a buffer without the overhead of full object allocation. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total number of bytes the object occupies in the buffer, including all variable-length fields. |
| clone() | TriggerCooldownInteraction | O(N) | Creates a deep copy of the object. Useful on the server-side for creating variations of a template interaction. |

## Integration Patterns

### Standard Usage

The canonical use case is within a client-side network handler. The handler receives a buffer, identifies the packet type, and uses the static `deserialize` method to construct the object before passing it to a downstream game system.

```java
// In a client-side network packet handler...
ByteBuf incomingPacket = ...;
int dataOffset = ...; // Start of the interaction data in the buffer

// Validate structure before full deserialization for security and stability
ValidationResult result = TriggerCooldownInteraction.validateStructure(incomingPacket, dataOffset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid TriggerCooldownInteraction: " + result.error());
}

// Deserialize the data into a concrete object
TriggerCooldownInteraction interaction = TriggerCooldownInteraction.deserialize(incomingPacket, dataOffset);

// Pass the resulting data object to the appropriate game system for processing
// The interaction object should not be modified past this point.
InteractionSystem interactionSystem = client.getInteractionSystem();
interactionSystem.applyInteractionDefinition(interaction);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation on Client:** Do not use `new TriggerCooldownInteraction()` on the client. Client-side instances must only be created via deserialization from a server packet.
-   **State Mutation After Deserialization:** Do not modify the fields of a deserialized object. Game systems should treat it as a read-only data record. Modifying it can break game state assumptions.
-   **Cross-Thread Sharing:** Never pass an instance of this object to another thread after creation. If data must be shared, extract the primitive values into a thread-safe structure.

## Data Pipeline

The flow of this object is unidirectional from server to client. It originates as a server-side configuration object and terminates as a client-side data record used to influence game logic.

> Flow:
> Server Game Logic -> **TriggerCooldownInteraction Instance** -> `serialize()` -> Netty ByteBuf -> Network Transmission -> Client Netty ByteBuf -> `deserialize()` -> **TriggerCooldownInteraction Instance** -> Client Interaction System -> Game State Update

