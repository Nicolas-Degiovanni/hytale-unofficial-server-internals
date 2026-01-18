---
description: Architectural reference for RunRootInteraction
---

# RunRootInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class RunRootInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The RunRootInteraction class is a specialized Data Transfer Object (DTO) operating within the Hytale network protocol layer. It represents a concrete, serializable command sent from the server to the client (or vice-versa) to initiate a complex, multi-stage player or entity action.

As a subclass of SimpleInteraction, it participates in the engine's interaction state machine. Its primary role is to encapsulate all the necessary parameters for a "root" or entry-point interaction, such as player movement modifiers, visual effects, conditional rules, and pointers to subsequent interactions in the chain (the *next* and *failed* fields).

The class design is heavily optimized for network performance. It employs a custom binary format consisting of two main parts:
1.  A **fixed-size block** containing primitive types and pointers.
2.  A **variable-size block** for nullable, complex objects like maps and arrays.

Offsets to the variable data are stored in the fixed block, allowing for fast, predictable reads of the core data while accommodating optional, larger payloads. A leading `nullBits` byte field acts as a bitmask to efficiently indicate which of the nullable, variable-sized fields are present in the payload, avoiding the need to transmit empty data.

## Lifecycle & Ownership
-   **Creation:** An instance of RunRootInteraction is created under two distinct circumstances:
    1.  **On the sending endpoint (e.g., Server):** Game logic instantiates it directly using its constructor, populating it with the parameters for the interaction that needs to be triggered on the remote peer.
    2.  **On the receiving endpoint (e.g., Client):** The network layer's packet dispatcher invokes the static `deserialize` method to construct an object from an incoming network `ByteBuf`. This is the most common creation path for objects that will be consumed by game logic.

-   **Scope:** The object's lifetime is ephemeral and tied to the processing of a single network packet. It is created, its data is consumed by a handler, and it is then immediately eligible for garbage collection. It holds no global state and is not intended to be cached or reused.

-   **Destruction:** Managed entirely by the Java Virtual Machine's garbage collector. There are no native resources or manual cleanup procedures associated with this object.

## Internal State & Concurrency
-   **State:** RunRootInteraction is a mutable data container. Its public fields are populated during deserialization or construction and are intended to be read by a single consumer. The state includes both primitive values and potentially deep object graphs for fields like `settings` and `effects`.

-   **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal locking or synchronization. It is designed to be created, deserialized, and processed within a single, well-defined thread, such as a Netty I/O thread or the main game logic thread. Concurrent modification from multiple threads will lead to race conditions and undefined behavior.

    **WARNING:** Do not share instances of this class across threads without implementing external, explicit synchronization. The standard pattern is to process it fully on the network thread or pass it to another single-threaded consumer via a thread-safe queue.

## API Surface
The public contract is primarily focused on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static RunRootInteraction | O(N) | Constructs a new object by reading from a network buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided network buffer. Returns the number of bytes written. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural integrity check on the buffered data without full deserialization. Crucial for security and stability. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a serialized object within a buffer. Used by the network layer to advance the buffer's read index. |
| computeSize() | int | O(N) | Pre-calculates the size in bytes that the object will occupy when serialized. |
| clone() | RunRootInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The object is typically deserialized by a central packet handler and dispatched to a system responsible for managing player interactions.

```java
// Executed on the network thread by a packet dispatcher
void handlePacket(ByteBuf packetBuffer) {
    // 1. Validate before processing to prevent malformed packet exploits
    ValidationResult result = RunRootInteraction.validateStructure(packetBuffer, packetBuffer.readerIndex());
    if (!result.isValid()) {
        throw new ProtocolException("Invalid RunRootInteraction: " + result.error());
    }

    // 2. Deserialize into a concrete object
    RunRootInteraction interaction = RunRootInteraction.deserialize(packetBuffer, packetBuffer.readerIndex());

    // 3. Advance the buffer's reader index
    int bytesConsumed = RunRootInteraction.computeBytesConsumed(packetBuffer, packetBuffer.readerIndex());
    packetBuffer.skipBytes(bytesConsumed);

    // 4. Pass the immutable data to the game logic thread for processing
    gameLogicExecutor.submit(() -> {
        playerInteractionSystem.startInteraction(interaction);
    });
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. Deserializing untrusted, unvalidated data can lead to out-of-memory errors, security vulnerabilities, or server crashes.
-   **Direct Instantiation on Receiver:** Do not use `new RunRootInteraction()` on the receiving end of the network. The only correct way to create an instance from network data is via the static `deserialize` method.
-   **State Modification after Deserialization:** Do not modify the state of a deserialized RunRootInteraction object. Treat it as an immutable record for the duration of its processing to prevent unexpected side effects. If modification is needed, use the `clone()` method first.

## Data Pipeline
The primary flow involves the deserialization of network data into a structured object that the game engine can understand and act upon.

> Flow:
> Inbound Network ByteBuf -> Protocol Dispatcher -> **RunRootInteraction.validateStructure()** -> **RunRootInteraction.deserialize()** -> Game Interaction System -> Player State Update

