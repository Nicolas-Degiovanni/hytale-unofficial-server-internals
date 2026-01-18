---
description: Architectural reference for ChangeStatInteraction
---

# ChangeStatInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class ChangeStatInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The ChangeStatInteraction class is a concrete Data Transfer Object (DTO) that represents a specific game action: the modification of an entity's statistics. It serves as the canonical, language-agnostic data contract between the client and server for this operation. As a subclass of SimpleInteraction, it fits within a larger framework of game events that can be triggered and processed by the core game loop.

Its primary architectural role is to facilitate the serialization and deserialization of stat-change events to and from the network byte stream. The class is designed for high-performance network I/O and employs a custom binary layout to minimize payload size.

This layout consists of two main parts:
1.  **Fixed-Size Block:** A 46-byte header containing primitive fields and offsets to variable-sized data. This allows for constant-time access to core properties.
2.  **Variable-Size Block:** A subsequent data region containing complex types like maps and arrays (e.g., statModifiers, tags).

A single leading byte, the *nullable bit field*, acts as a bitmask to indicate the presence or absence of optional, variable-sized fields. This is a critical optimization that avoids transmitting unnecessary data over the network.

## Lifecycle & Ownership
-   **Creation:** An instance is created under two primary scenarios:
    1.  **Inbound:** The protocol layer invokes the static `deserialize` factory method to construct an object from an incoming network ByteBuf.
    2.  **Outbound:** Game logic instantiates a new object via its constructor, populates its fields, and submits it to the network layer for serialization.
-   **Scope:** Transient. The object's lifetime is typically confined to the scope of a single network packet processing cycle or game tick. It is created, used immediately by a handler or system, and then becomes eligible for garbage collection.
-   **Destruction:** There is no manual destruction. The Java Garbage Collector reclaims the object's memory once it is no longer referenced.

## Internal State & Concurrency
-   **State:** Mutable. All public fields can be directly modified after instantiation. This class is a simple data container and holds no internal logic beyond serialization. It does not cache data.
-   **Thread Safety:** **WARNING:** This object is not thread-safe. It contains no internal locking or synchronization primitives. It is designed to be created and processed within a single thread, such as a Netty I/O worker or the main game thread. Concurrent access from multiple threads without external synchronization will lead to race conditions and undefined behavior.

## API Surface
The public contract is dominated by static methods for validation and deserialization from a raw buffer, and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ChangeStatInteraction | O(N) | **[Factory]** Constructs an instance from a binary representation in a buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into a binary representation in the provided buffer. Returns bytes written. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a security and integrity check on the binary data *before* attempting full deserialization. |
| computeSize() | int | O(N) | Calculates the total number of bytes the object will consume when serialized. |
| clone() | ChangeStatInteraction | O(N) | Creates a deep copy of the interaction object and its contained data structures. |

*N refers to the total size of the variable-length fields within the payload.*

## Integration Patterns

### Standard Usage
This object should be treated as an immutable message after deserialization or before serialization. The typical flow involves deserializing from the network, passing the object to a handler, and applying the state change to the game world.

```java
// Example: In a network packet handler
// The ByteBuf contains the raw packet data.
ChangeStatInteraction interaction = ChangeStatInteraction.deserialize(buf, offset);

// Dispatch to a game system for processing
gameWorld.getStatSystem().applyInteraction(interaction);
```

### Anti-Patterns (Do NOT do this)
-   **Object Reuse:** Do not reuse a ChangeStatInteraction instance across multiple network events or game ticks. They are cheap to create and should be treated as single-use, transient objects.
-   **Concurrent Modification:** Do not read properties from one thread while another thread is populating the object for serialization. All operations on a given instance must be confined to a single thread.
-   **Ignoring Validation:** Do not call `deserialize` on untrusted data without first calling `validateStructure`. Bypassing validation exposes the server to malformed packets that could trigger exceptions or crashes.

## Data Pipeline
The class is a key component in the data flow between the network stack and the game logic. The following illustrates the inbound data pipeline for a received packet.

> Flow:
> Raw Network ByteBuf -> Protocol Decoder -> **ChangeStatInteraction.deserialize()** -> ChangeStatInteraction Instance -> Game Event Bus -> Stat Processing System -> Entity Component State Change

