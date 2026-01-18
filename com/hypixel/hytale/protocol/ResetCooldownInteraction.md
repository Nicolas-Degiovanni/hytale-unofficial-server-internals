---
description: Architectural reference for ResetCooldownInteraction
---

# ResetCooldownInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ResetCooldownInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The ResetCooldownInteraction class is a specialized Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or manager, but rather a structured container for data that represents a specific game event: the modification or resetting of an action cooldown.

As a subclass of SimpleInteraction, it forms part of a larger polymorphic system for defining game interactions. Its primary responsibility is to facilitate the transfer of complex interaction state between the server and client.

The most critical architectural feature is its custom binary serialization format. To achieve high performance and minimize network bandwidth, it employs a sophisticated layout:

1.  **Nullability Bitfield:** The first byte of the serialized data is a bitmask, where each bit indicates whether a corresponding nullable field is present in the data stream. This avoids the need to serialize null markers for every optional field.
2.  **Fixed-Size Data Block:** A block of fixed-size fields (integers, floats, booleans) is located at the beginning of the structure for fast, direct-offset access.
3.  **Variable-Size Data Block:** Complex, variable-length data types (such as maps, arrays, and nested objects) are appended after the fixed block. The fixed block contains integer offsets pointing to the start of each variable field within this block.

This design allows for extremely efficient parsing, as the decoder can read the fixed block, determine which variable fields exist from the bitfield, and then jump directly to their data without sequential scanning.

## Lifecycle & Ownership

-   **Creation:** Instances are created under two primary circumstances:
    1.  **Receiving End:** The network protocol decoder instantiates the object via the static `deserialize` factory method when an incoming network packet of the corresponding type is processed.
    2.  **Sending End:** Game logic on the server (or client) instantiates the object using its constructor, populates its fields, and hands it to the protocol encoder for serialization.

-   **Scope:** The lifetime of a ResetCooldownInteraction instance is exceptionally short and tied to the processing of a single network packet or game event. It is a transient object, not intended to be stored or referenced long-term.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once it has been processed by the relevant game logic handler (e.g., updating a player's state), it falls out of scope and becomes eligible for collection. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** The object's state is entirely mutable through its public fields. It is designed as a simple data container whose fields are populated once during creation or deserialization.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game loop thread. Modifying or accessing an instance from multiple threads without explicit external synchronization will result in data corruption and undefined behavior.

    **WARNING:** Never share an instance of this class across threads. If state must be passed to another thread, create a deep copy using the `clone` method or extract the data into a thread-safe structure.

## API Surface

The public contract is dominated by static methods for serialization and validation, reflecting its role in the protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ResetCooldownInteraction | O(N) | Constructs an object by reading from a ByteBuf. The primary factory method on the receiving end. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into a ByteBuf. Returns the number of bytes written. Throws ProtocolException if data constraints are violated. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object within a buffer without performing a full deserialization. Used for efficient buffer parsing. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a structural validation of the raw byte data before attempting deserialization. Crucial for preventing crashes from malformed or malicious packets. |
| clone() | ResetCooldownInteraction | O(N) | Creates a deep copy of the object and its nested data structures. |

## Integration Patterns

### Standard Usage

The object is almost always handled within a network packet handler. The handler deserializes the interaction from the buffer and passes it to the game logic for processing.

```java
// Example within a network packet handler
void handlePacket(ByteBuf packetBuffer) {
    // Validate structure before attempting to deserialize
    ValidationResult result = ResetCooldownInteraction.validateStructure(packetBuffer, 0);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid ResetCooldownInteraction: " + result.error());
    }

    // Deserialize and process
    ResetCooldownInteraction interaction = ResetCooldownInteraction.deserialize(packetBuffer, 0);
    player.getInteractionController().processInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not hold references to this object in long-lived components. Its purpose is transient. If you need to persist its data, copy the values into a dedicated state object.
-   **Mutation After Serialization:** Do not modify the state of a ResetCooldownInteraction object after it has been passed to the network layer for serialization. The behavior is undefined and may lead to sending corrupted data.
-   **Ignoring Validation:** Bypassing `validateStructure` in production environments is dangerous. Always validate incoming data from an external source before attempting to deserialize it to prevent potential buffer overflows or denial-of-service vulnerabilities.

## Data Pipeline

The ResetCooldownInteraction acts as a data payload that flows through the network stack and into the game engine.

**Inbound Flow (Server -> Client):**
> Flow:
> Network ByteBuf -> Protocol Decoder -> **ResetCooldownInteraction.deserialize()** -> Game Logic Handler -> Player Cooldown State Update

**Outbound Flow (Server -> Client):**
> Flow:
> Game Event Trigger -> **new ResetCooldownInteraction()** -> Protocol Encoder -> **interaction.serialize()** -> Network ByteBuf

