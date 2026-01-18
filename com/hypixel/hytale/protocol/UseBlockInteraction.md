---
description: Architectural reference for UseBlockInteraction
---

# UseBlockInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class UseBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The UseBlockInteraction class is a data contract object, not a service. It serves as a high-fidelity representation of a "use block" interaction within the Hytale network protocol. Its primary responsibility is to encapsulate the complex set of rules, effects, and state transitions that occur when a player interacts with a specific block in the world.

This class is architected for extreme performance and minimal overhead during network transmission. It achieves this through a custom binary serialization format that is tightly coupled with the Netty ByteBuf component. The binary layout consists of two main parts:

1.  **Fixed-Size Block:** A 40-byte header containing primitive fields and offsets. This allows for constant-time access to core data and the structure of the rest of the payload.
2.  **Variable-Size Block:** A subsequent data region containing complex, nullable objects like maps, arrays, and nested structures. The location of each variable field is specified by an integer offset within the fixed-size block.

A critical feature of this design is the initial `nullBits` byte field. This bitmask efficiently encodes the presence or absence of the five possible variable-sized fields, allowing the deserializer to skip reading entire sections of the payload if they are not present. This "struct-of-pointers" approach is fundamental to the protocol's efficiency, enabling rapid parsing and validation.

## Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer (e.g., Server):** Instantiated directly via its constructor (`new UseBlockInteraction(...)`) by game logic systems that define the interaction's properties.
    -   **Receiving Peer (e.g., Client):** Instantiated exclusively by the static factory method `UseBlockInteraction.deserialize(buf, offset)`. This occurs deep within the network protocol decoding pipeline when an incoming packet of this type is identified.

-   **Scope:** The lifetime of a UseBlockInteraction instance is exceptionally short and bound to a single network transaction. It is created, serialized, transmitted, and then becomes eligible for garbage collection. On the receiving end, it is deserialized, its data is consumed by the relevant game system, and it is immediately discarded. **It is not intended for long-term storage.**

-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
-   **State:** The UseBlockInteraction is a **mutable** data container. Its fields are directly accessible and are populated once during deserialization or construction. The existence of a `clone` method indicates that creating independent, deep copies of its state is a supported and sometimes necessary operation.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed for use within the confines of a single network thread (for serialization/deserialization) or a single game update thread (for processing). Modifying an instance from multiple threads will result in data corruption.

    **WARNING:** The serialization and deserialization methods operate on Netty ByteBufs, which have a strict ownership and reference counting model. Mishandling the buffer in a multithreaded context can lead to memory leaks or illegal access exceptions.

## API Surface
The public contract is dominated by static methods for interacting with raw byte buffers, reinforcing its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static UseBlockInteraction | O(N) | Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state to a ByteBuf. Returns the number of bytes written. Throws ProtocolException on constraint violations. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(V) | Performs a security and integrity check on the buffer's contents without a full deserialization. V is the number of variable fields present. |
| computeBytesConsumed(ByteBuf, int) | static int | O(V) | Calculates the total size of the serialized object within a buffer. Used by higher-level parsers to advance the buffer reader index. |
| clone() | UseBlockInteraction | O(N) | Creates a deep copy of the object and all its nested data structures. |

*Complexity Note: N refers to the total size of the serialized data. V refers to the number of non-null variable-sized fields.*

## Integration Patterns

### Standard Usage
The class is intended to be used within a network packet handler. The handler identifies the packet type, validates its structure, and then deserializes it into an object for the game logic layer.

```java
// Example from a network packet handler
ValidationResult result = UseBlockInteraction.validateStructure(packetBuffer, offset);
if (!result.isValid()) {
    // Disconnect client or log error
    throw new ProtocolException("Invalid UseBlockInteraction: " + result.error());
}

UseBlockInteraction interaction = UseBlockInteraction.deserialize(packetBuffer, offset);
gameLogic.processPlayerInteraction(player, interaction);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Deserialization:** Do not use `new UseBlockInteraction()` on the receiving end and attempt to populate its fields manually by reading from the buffer. The binary layout is complex, and this will inevitably lead to parsing errors. Always use the static `deserialize` method.
-   **Ignoring Validation:** On a server receiving data from an untrusted client, failing to call `validateStructure` before `deserialize` is a critical security vulnerability. A malicious client could send a malformed packet (e.g., with an invalid length field) that causes excessive memory allocation or crashes the server, leading to a denial-of-service.
-   **Instance Caching or Re-use:** Do not cache or reuse UseBlockInteraction instances between packets. They are lightweight objects, and attempting to reuse them can introduce severe state corruption bugs that are difficult to diagnose. Create a new instance for each message.

## Data Pipeline
The UseBlockInteraction acts as a data payload that flows from the game simulation on one peer to the game simulation on another.

> **Serialization Flow (Server -> Client):**
> Game Logic Defines Interaction -> `new UseBlockInteraction(...)` -> **`serialize(buf)`** -> Netty Channel Pipeline -> Raw TCP Packet

> **Deserialization Flow (Client <- Server):**
> Raw TCP Packet -> Netty Channel Pipeline -> **`validateStructure(buf)`** -> **`deserialize(buf)`** -> Game Logic Consumes Data -> World State Update

