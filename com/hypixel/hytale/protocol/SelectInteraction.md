---
description: Architectural reference for SelectInteraction
---

# SelectInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class SelectInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The SelectInteraction class is a data transfer object (DTO) that represents a specific type of game interaction within the Hytale network protocol. It defines a state in the game's interaction graph where the outcome depends on a player's selection of a target, such as an entity or block.

Its primary role is to serialize and deserialize this state for transmission between the client and server. The class employs a highly optimized, custom binary format to minimize network bandwidth. This format is composed of two main parts:

1.  **Fixed-Size Block:** A 25-byte block at the beginning of the payload containing primitive fields like identifiers, flags, and floating-point values. This ensures predictable and fast access to core data.
2.  **Variable-Size Data Block:** A subsequent block containing complex, nullable data structures like lists, maps, and nested objects.

A sophisticated offset-based system links these two blocks. The payload begins with a single byte, a **bitmask** referred to as `nullBits`, which indicates which of the optional, variable-sized fields are present in the stream. Following the fixed block is a table of 4-byte integer offsets. For each non-null complex field, its corresponding entry in this table points to the start of its data within the variable-size block. This design avoids wasting space for null fields and allows for efficient, non-sequential parsing of the data stream.

## Lifecycle & Ownership
-   **Creation:** An instance of SelectInteraction is created under two conditions:
    1.  By the network layer when the static `deserialize` method is invoked on an incoming `ByteBuf` from a Netty pipeline.
    2.  By game logic constructing a new interaction to be sent over the network, typically using the all-arguments constructor.
-   **Scope:** The object is transient and has a very short lifetime. It exists only for the duration of its processing within a single network event or game tick.
-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after the network packet has been processed or sent. There are no manual cleanup requirements.

## Internal State & Concurrency
-   **State:** The class is fundamentally a mutable data container. All fields are public, allowing for direct, low-overhead manipulation. It holds the complete state for a selection-based interaction, including rules, visual effects, and pointers to subsequent interaction states (`next`, `failed`).
-   **Thread Safety:** **WARNING:** This class is not thread-safe. It is designed for performance within a single-threaded context, such as a Netty event loop or the main game thread. Concurrent access from multiple threads will lead to race conditions and data corruption. All operations on a SelectInteraction instance must be externally synchronized or, preferably, confined to a single thread.

## API Surface
The public contract is dominated by static methods for serialization and validation, reflecting its role as a protocol message definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static SelectInteraction | O(N) | Constructs a new instance by reading from a Netty byte buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided byte buffer. Returns the number of bytes written. |
| computeSize() | int | O(N) | Calculates the total number of bytes required to serialize the object. Useful for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural integrity check on the binary data in a buffer without performing a full deserialization. |
| clone() | SelectInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol dispatcher. The dispatcher reads a packet identifier, determines the type is SelectInteraction, and invokes the static `deserialize` method. The resulting object is then passed to a higher-level game system for processing.

```java
// Executed within a Netty channel handler or equivalent
ByteBuf packetPayload = ...;
ValidationResult result = SelectInteraction.validateStructure(packetPayload, 0);

if (result.isValid()) {
    SelectInteraction interaction = SelectInteraction.deserialize(packetPayload, 0);
    // Pass the fully hydrated object to the game's interaction engine
    gameInteractionSystem.processInteraction(interaction);
} else {
    // Handle or log the corrupt packet
    throw new ProtocolException("Invalid SelectInteraction: " + result.error());
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not retain instances of SelectInteraction beyond the immediate scope of processing. They are not designed to be part of the persistent game state.
-   **Cross-Thread Sharing:** Never pass an instance between threads without deep cloning or proper synchronization. The mutable, non-thread-safe design makes this extremely hazardous.
-   **Manual Field Population:** While possible, creating an empty instance with `new SelectInteraction()` and setting fields manually is error-prone. The all-arguments constructor should be preferred when creating new instances in code to ensure all required fields are initialized.

## Data Pipeline
SelectInteraction acts as a data model within the network protocol pipeline. It represents the deserialized state of a message as it flows from the network hardware to the game logic.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> Packet Framer/Decoder -> **SelectInteraction.deserialize** -> Game Interaction System -> World State Update

