---
description: Architectural reference for StatsConditionInteraction
---

# StatsConditionInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Protocol Data Object)

## Definition
```java
// Signature
public class StatsConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The StatsConditionInteraction class is a specialized Data Transfer Object (DTO) operating within the Hytale Protocol Layer. It does not represent a service or manager, but rather a concrete data definition for a node in the game's **Interaction Graph**. Its primary function is to serialize and deserialize the properties of a conditional interaction that evaluates an entity's statistics (e.g., health, mana, stamina).

This class is a critical component for defining dynamic gameplay behavior that can be configured by designers and synchronized between the client and server. It encapsulates the logic for checking if a character's stat meets a certain threshold, and what should happen (which interaction node to transition to) on success or failure.

The binary format is highly optimized for network performance, employing a custom serialization strategy:
1.  **Null Bitmask:** A single byte, `nullBits`, at the start of the payload acts as a bitfield. Each bit corresponds to a nullable, variable-sized field (e.g., `effects`, `costs`). This avoids wasting bytes on null markers for every optional field.
2.  **Fixed-Size Block:** A contiguous block of memory (`FIXED_BLOCK_SIZE`) holds all fixed-size primitive types like booleans, floats, and integers.
3.  **Variable-Size Block:** Following the fixed block, a variable-sized data region contains the serialized data for complex types like maps, arrays, and nested objects. The fixed-size block contains integer offsets pointing to the start of each corresponding object within this variable region. This structure prevents parsing the entire object to find a specific field and allows for efficient validation.

## Lifecycle & Ownership
-   **Creation:** An instance of StatsConditionInteraction is created under two distinct circumstances:
    1.  **Deserialization (Primary):** The static factory method `deserialize` is invoked by the network protocol decoder when a corresponding packet is read from a Netty ByteBuf. This is the standard creation path on the receiving end (client or server).
    2.  **Serialization:** On the sending end, an instance is constructed, populated with game data (e.g., from a content file or game state), and then passed to a serializer which invokes the `serialize` method.
-   **Scope:** The object's lifetime is extremely short. It is scoped to the processing of a single network packet or the evaluation of one step in the game's interaction logic. It is not designed to be a long-lived component.
-   **Destruction:** As a simple data object with no external resource handles, it is managed entirely by the Java Garbage Collector. It becomes eligible for collection as soon as it falls out of the scope of the packet handler or game logic method that was using it.

## Internal State & Concurrency
-   **State:** The object is **mutable**. Its public fields can be modified after creation. It is designed as a simple container for data, either freshly deserialized from the network or being prepared for serialization. There is no internal caching; the fields directly represent the object's state.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit external synchronization. It is intended for use within the single-threaded context of a Netty event loop or the main game logic thread. The protocol layer's design guarantees that a single instance is processed sequentially, mitigating concurrency risks.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a protocol data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static StatsConditionInteraction | O(N) | Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state to a ByteBuf and returns the number of bytes written. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Scans a ByteBuf to verify data integrity without full deserialization. Critical for security. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object already present in a buffer. |
| computeSize() | int | O(N) | Calculates the required buffer size for serializing the current in-memory object. |
| clone() | StatsConditionInteraction | O(N) | Performs a deep copy of the object and its contained data structures. |

*N represents the number of elements in variable-sized fields like maps and arrays.*

## Integration Patterns

### Standard Usage
This class is not intended for direct use by high-level gameplay programmers. It is an implementation detail of the protocol layer. A system responsible for processing interactions would receive this object after it has been deserialized.

```java
// Hypothetical usage inside a low-level packet handler
// The buffer and offset are provided by the network stack

if (InteractionType.fromId(id) == InteractionType.STATS_CONDITION) {
    ValidationResult result = StatsConditionInteraction.validateStructure(buffer, offset);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid StatsConditionInteraction: " + result.error());
    }

    StatsConditionInteraction interaction = StatsConditionInteraction.deserialize(buffer, offset);
    
    // Pass the deserialized, validated object to the game logic engine
    game.getInteractionProcessor().evaluate(player, interaction);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the state of a deserialized StatsConditionInteraction object. This breaks the "source of truth" principle, as the in-memory state will no longer match what was sent over the network, leading to hard-to-debug desynchronization bugs. Treat deserialized instances as immutable.
-   **Ignoring Validation:** Never call `deserialize` on data from an untrusted source (e.g., a client packet) without first calling `validateStructure`. Bypassing this check exposes the server to denial-of-service attacks via maliciously crafted packets that could cause `OutOfMemoryError` or other unhandled exceptions.
-   **Manual Instantiation:** Avoid using `new StatsConditionInteraction()` in general game code. These objects should be defined in content files and loaded by a dedicated system that handles their creation and serialization.

## Data Pipeline
StatsConditionInteraction serves as a data container at a specific point in the network and game logic pipeline. It translates raw bytes into a structured, usable format for the game engine.

> Flow:
> Network Byte Stream -> Netty Channel Handler -> Protocol Packet Splitter -> **StatsConditionInteraction.deserialize** -> In-Memory Object -> Interaction Graph Processor -> Game State Mutation

