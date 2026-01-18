---
description: Architectural reference for PickBlockInteraction
---

# PickBlockInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class PickBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The PickBlockInteraction class is a concrete data structure representing a specific type of game interaction, likely triggered when a player uses the "pick block" control. It serves as a Data Transfer Object, designed for efficient serialization and deserialization from a binary stream, such as a network packet or a game asset file. This class does not contain any logic; it is a pure data container.

Its primary architectural function is to define the complete set of properties and behaviors for a block-picking action. This includes visual effects, conditional rules, camera modifications, and state transitions within an interaction graph (represented by the *next* and *failed* fields).

The binary layout is highly optimized for performance and size, employing a hybrid fixed-size and variable-size block structure:

1.  **Fixed-Size Block (40 bytes):** Contains primitive fields and an offset table for variable-sized data. This allows for extremely fast access to core properties without parsing the entire object.
2.  **Nullable Bit Field (1 byte):** The first byte of the payload is a bitmask indicating which of the nullable, complex objects (like *effects* or *rules*) are present in the stream. This avoids wasting space for null data.
3.  **Variable-Size Block:** Appended after the fixed block, this section contains the actual data for the complex objects. The offset table in the fixed block points to the start of each object within this variable section.

This design pattern is common in high-performance network protocols, minimizing both bandwidth and CPU time required for processing game data.

## Lifecycle & Ownership

-   **Creation:** Instances of PickBlockInteraction are almost exclusively created by the static `deserialize` method. This occurs when the protocol layer reads a binary payload from a Netty ByteBuf and materializes it into a Java object. Manual instantiation via its constructor is reserved for internal tooling or server-side content authoring.
-   **Scope:** The object is transient. Its lifetime is typically bound to the scope of a single network packet handler or a game logic update tick. It is read, processed by a consuming system (e.g., an InteractionManager), and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency

-   **State:** The class is fully mutable. All fields are accessible and can be modified after creation. However, in practice, it should be treated as an immutable object after deserialization to prevent unforeseen side effects in game systems.
-   **Thread Safety:** **This class is not thread-safe.** It is a simple POJO with no internal locking or synchronization.

    **WARNING:** Concurrent reads and writes will lead to race conditions and undefined behavior. Instances must not be shared across threads without explicit external synchronization. Its intended use is within a single-threaded context, such as a Netty event loop or the main game thread.

## API Surface

The public contract is dominated by static methods for serialization, deserialization, and validation, operating directly on Netty's ByteBuf.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | PickBlockInteraction | O(N) | **(Static)** Reads from the buffer at an offset and constructs a new instance. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided buffer. Returns the number of bytes written. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | **(Static)** Calculates the total size of a serialized object in a buffer without full deserialization. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **(Static)** Performs a structural integrity check on the binary data without creating an object. Crucial for security and preventing parsing errors. |
| clone() | PickBlockInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage

The class is designed to be used by the network or asset loading layer. A handler receives a buffer, deserializes the object, and passes it to a game system for processing.

```java
// In a network handler or asset loader
ByteBuf buffer = ... // incoming data
int offset = ... // position of the object in the buffer

// Validate before deserializing to prevent errors and exploits
ValidationResult result = PickBlockInteraction.validateStructure(buffer, offset);
if (!result.isValid()) {
    throw new SecurityException("Invalid PickBlockInteraction data: " + result.error());
}

// Deserialize and pass to game logic
PickBlockInteraction interaction = PickBlockInteraction.deserialize(buffer, offset);
game.getInteractionSystem().processInteraction(player, interaction);
```

### Anti-Patterns (Do NOT do this)

-   **Manual Construction:** Do not use `new PickBlockInteraction()` for game logic. The object's state is complex and should be defined in data files and loaded via the deserialization pipeline. Manual creation is error-prone.
-   **State Mutation After Deserialization:** Avoid modifying the object after it has been deserialized. Downstream systems may expect the data to be immutable. If modification is needed, use the `clone()` method to create a new instance.
-   **Ignoring Validation:** Never call `deserialize` on untrusted data without first calling `validateStructure`. Failing to do so can result in uncaught exceptions, buffer over-reads, or other vulnerabilities.

## Data Pipeline

PickBlockInteraction acts as a bridge, converting low-level binary data into a high-level, structured representation for use by the game engine.

> Flow:
> Network ByteBuf -> **PickBlockInteraction.deserialize** -> **PickBlockInteraction Instance** -> InteractionSystem -> Game State Update

