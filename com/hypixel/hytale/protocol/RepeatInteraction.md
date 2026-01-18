---
description: Architectural reference for RepeatInteraction
---

# RepeatInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class RepeatInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The RepeatInteraction class is a Data Transfer Object (DTO) that defines a specific type of game interaction within the Hytale network protocol. It represents a node in a larger interaction graph, which functions as a state machine for player or entity actions. This class extends SimpleInteraction, inheriting a baseline of interaction properties, and adds functionality for looping or repeating an action.

Its primary role is to be serialized into a binary format for network transmission and deserialized back into an object on the receiving end. The binary layout is highly optimized for size, employing a combination of a fixed-size data block for primitive fields and a variable-size block for complex or nullable objects. A `nullBits` bitfield is used as a header to indicate which of the optional, variable-sized fields are present in the payload, avoiding the need to transmit empty data.

This object does not contain any logic; it is a pure data container that describes the properties of an interaction, such as its effects, rules, and transitions to subsequent interactions upon success, failure, or branching.

## Lifecycle & Ownership
- **Creation:** Instances of RepeatInteraction are created in two primary scenarios:
    1.  By a network protocol decoder (e.g., a Netty ChannelInboundHandler) which calls the static `deserialize` method to construct an object from an incoming ByteBuf.
    2.  By server-side game logic when defining a new interaction to be sent to a client. In this case, the constructor is used directly to populate the object before serialization.

- **Scope:** The object is ephemeral and has a very short lifecycle. It is scoped to the transaction in which it is processed. An instance created from a network packet exists only long enough for the game logic to read its data and update the relevant game state.

- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as no more references point to it, which is typically immediately after the processing of the network packet or game event is complete. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The state of a RepeatInteraction instance is entirely mutable. All of its fields are public and can be modified after construction. This design facilitates easy population of the object before serialization or for debugging purposes. The object itself does not manage or cache any external state.

- **Thread Safety:** **This class is not thread-safe.** It is a simple data container with no internal locking or synchronization. Instances must be confined to a single thread. Typically, this will be a Netty event loop thread for deserialization or the main game thread for logic processing.

    **Warning:** Sharing a RepeatInteraction instance across multiple threads without explicit external synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections that are accessed concurrently.

## API Surface
The primary contract of this class revolves around its static serialization and validation utilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | RepeatInteraction | O(N) | **[Primary Entry Point]** Constructs an object by reading from a binary buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state to a binary buffer. Returns the number of bytes written. Throws ProtocolException if data constraints are violated. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Calculates the total size of a serialized object within a buffer without full deserialization. Used for advancing buffer read pointers. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a structural integrity check on the binary data without creating an object. Crucial for security and preventing malformed packet exploits. |
| clone() | RepeatInteraction | O(N) | Creates a deep copy of the interaction object. |

## Integration Patterns

### Standard Usage
This class is intended to be used by the network protocol layer. A codec or handler will deserialize the object from a buffer and pass it to a consumer, such as an interaction state machine or event bus.

```java
// Executed within a Netty channel handler or similar context
public void processInteractionData(ByteBuf networkBuffer) {
    // Validate before deserializing to prevent errors
    ValidationResult result = RepeatInteraction.validateStructure(networkBuffer, networkBuffer.readerIndex());
    if (!result.isValid()) {
        throw new ProtocolException("Invalid RepeatInteraction packet: " + result.error());
    }

    // Deserialize and pass to game logic
    RepeatInteraction interaction = RepeatInteraction.deserialize(networkBuffer, networkBuffer.readerIndex());
    int bytesConsumed = RepeatInteraction.computeBytesConsumed(networkBuffer, networkBuffer.readerIndex());
    networkBuffer.skipBytes(bytesConsumed);

    // The 'interaction' object is now ready for processing by the game engine
    gameState.handleInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to RepeatInteraction instances in long-lived services or caches. They represent a point-in-time snapshot of data and should be processed immediately. Storing them can lead to memory leaks and stale state.

- **Concurrent Modification:** Never modify an instance from multiple threads. If an interaction needs to be processed by another thread, create a deep copy using the `clone` method for the new thread.

- **Manual Deserialization:** Avoid reading fields from the ByteBuf manually. Always use the provided `deserialize` and `computeBytesConsumed` methods to ensure forward compatibility and correctness, especially given the complex variable-offset data layout.

## Data Pipeline
The RepeatInteraction object is a payload carrier in the network data flow. It serves as the bridge between the raw binary protocol and the structured game logic.

> **Inbound Flow:**
> Network ByteBuf -> Protocol Decoder -> **RepeatInteraction.deserialize()** -> In-Memory RepeatInteraction Instance -> Game Logic / State Machine -> World State Update

> **Outbound Flow:**
> Game Logic Defines Interaction -> New RepeatInteraction Instance -> **RepeatInteraction.serialize()** -> Protocol Encoder -> Network ByteBuf

