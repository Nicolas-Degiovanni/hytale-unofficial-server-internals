---
description: Architectural reference for ChargingInteraction
---

# ChargingInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ChargingInteraction extends Interaction {
```

## Architecture & Concepts
The ChargingInteraction class is a specialized data model that defines the behavior of a time-based, "charge-up" player action. Examples include drawing a bow, charging a magical spell, or preparing a heavy melee attack. It serves as a Data Transfer Object (DTO) within the Hytale network protocol, designed exclusively for the transmission of interaction state between the server and client.

Its architecture is optimized for network efficiency. The serialization format is a custom binary layout composed of two main parts:
1.  **Fixed-Size Block:** A 47-byte block containing frequently used, non-nullable fields for predictable and fast reads.
2.  **Variable-Size Block:** A subsequent data block for optional or variable-length fields like maps and arrays. Pointers (integer offsets) within the fixed block reference the start of each data structure in the variable block.

A critical component of this design is the **nullBits** bitmask. This is the first byte of the serialized object, where each bit corresponds to a nullable field. If a bit is set, the corresponding field is present in the payload, and its offset is valid. This avoids transmitting unnecessary data for optional fields, significantly reducing packet size.

This class is a passive data container; it contains no logic related to the execution of the interaction itself. That responsibility lies with higher-level game systems which interpret the data within this object.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the network layer through the static `deserialize` factory method when a packet containing this interaction data is received. On the server, instances are created using the constructor, populated with data, and then passed to the network layer for serialization.
-   **Scope:** The object's lifetime is transient and context-bound. It exists only for the duration of processing a network packet and updating the relevant game state. It is not a long-lived object.
-   **Destruction:** As a standard Java object with no external resources, it is managed by the Garbage Collector. It becomes eligible for collection as soon as the game systems have finished processing its data and no references remain.

## Internal State & Concurrency
-   **State:** The ChargingInteraction is a highly mutable object. All of its fields are public, allowing direct modification after creation. Its state represents a complete snapshot of the configuration for a specific charging interaction as defined by the game's content scripts and server logic.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. Concurrent reads and writes from multiple threads without external locking mechanisms will result in data corruption and undefined behavior.

    **Warning:** Do not share instances of this class across threads. If multi-threaded processing is required, create a deep copy using the `clone` method for each thread.

## API Surface
The primary API contract revolves around serialization, deserialization, and validation of the network byte stream.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ChargingInteraction | O(N) | **Static Factory.** Constructs an instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into a ByteBuf according to the defined binary protocol. Returns the number of bytes written. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **Static Method.** Performs pre-flight checks on a ByteBuf to ensure data is well-formed before attempting deserialization. Critical for protocol security. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | **Static Method.** Calculates the total size of a serialized object in a buffer without performing a full deserialization. |
| clone() | ChargingInteraction | O(N) | Creates a deep copy of the object, including all nested collections and objects. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol handlers to decode incoming game data. The resulting object is then passed to game logic systems.

```java
// In a network packet handler or decoder...
ByteBuf packetData = ...;
int readOffset = ...;

// 1. (Optional but Recommended) Validate the data structure before parsing.
ValidationResult result = ChargingInteraction.validateStructure(packetData, readOffset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid ChargingInteraction: " + result.error());
}

// 2. Deserialize the byte stream into a usable object.
ChargingInteraction interaction = ChargingInteraction.deserialize(packetData, readOffset);

// 3. Pass the DTO to a game system for processing.
playerActionSystem.applyInteraction(player, interaction);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Avoid modifying the state of a deserialized object. Treat it as an immutable record for the current game tick. Modifying it can lead to desynchronization between game logic and the original server state.
-   **Bypassing Validation:** In production environments, never call `deserialize` without first calling `validateStructure`. Failing to do so can make the client vulnerable to crashes or exploits from malformed packets.
-   **Long-Term Storage:** Do not hold references to ChargingInteraction objects for longer than necessary. They are transient data carriers, not components of the persistent game state.

## Data Pipeline
The ChargingInteraction acts as a data contract for the network pipeline, translating raw bytes into a structured, usable format for the game engine.

> Flow:
> Network Packet (ByteBuf) -> **ChargingInteraction.validateStructure()** -> **ChargingInteraction.deserialize()** -> **ChargingInteraction** (Instance) -> Player Action System -> Game State Update

