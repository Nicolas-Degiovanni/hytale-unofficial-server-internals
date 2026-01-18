---
description: Architectural reference for ParallelInteraction
---

# ParallelInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ParallelInteraction extends Interaction {
```

## Architecture & Concepts

The ParallelInteraction class is a concrete implementation of a network protocol message, representing a game interaction that can execute concurrently with other player or world actions. It functions as a Data Transfer Object (DTO), designed for high-performance serialization and deserialization from a raw byte stream, specifically a Netty ByteBuf.

This class is a cornerstone of the client-server communication layer, defining a strict binary contract for how interaction data is structured on the wire. Its design eschews standard serialization frameworks in favor of a custom, manually-implemented binary format to achieve maximum control over memory layout and minimize processing overhead.

The core architectural pattern is a **hybrid fixed-variable block layout**:
1.  **Fixed Block:** The first 35 bytes of the serialized object contain a predictable structure of primitive types (floats, booleans) and, critically, integer offsets.
2.  **Nullable Bit Field:** The very first byte is a bitmask that indicates which of the nullable, variable-sized fields are present in the payload. This allows the deserializer to skip reading entire sections of data if they are not included, a significant performance optimization.
3.  **Variable Block:** Following the fixed block is a region containing the actual data for complex, variable-sized fields like lists, maps, and nested objects. The integer offsets in the fixed block point to the start of the corresponding data within this variable block.

This layout is optimized for read performance, allowing for direct memory access via offsets without needing to parse the entire message sequentially.

## Lifecycle & Ownership

-   **Creation:** An instance of ParallelInteraction is created under two primary circumstances:
    1.  **Deserialization:** The static factory method `deserialize` is called by a network pipeline handler (e.g., a Netty ChannelInboundHandler) when an incoming ByteBuf is identified as a ParallelInteraction message. This is the most common creation path.
    2.  **Direct Instantiation:** Game logic on the sending side (client or server) creates an instance via its constructor to define an interaction that needs to be transmitted. The object is populated with data and then passed to the serialization layer.

-   **Scope:** The object is transient and has a very short lifetime. It exists only for the duration of its processing within a single network event or game tick. It is not designed to be cached or held for long periods.

-   **Destruction:** The object holds no native resources and does not require explicit cleanup. It is managed entirely by the Java Garbage Collector and becomes eligible for collection as soon as no more references to it exist, typically after the network handler or game logic system has finished processing it.

## Internal State & Concurrency

-   **State:** The class is a mutable data container. All of its fields are public or package-private and can be modified after construction. It performs no internal validation on setters; data integrity is the responsibility of the creator or the `validateStructure` method.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be operated on by a single thread at a time, such as a Netty event loop thread. Concurrent modification of a ParallelInteraction instance from multiple threads will result in a corrupted state and unpredictable behavior. Any multi-threaded access must be protected by external synchronization mechanisms.

## API Surface

The primary contract of this class is its static methods for protocol handling, not its instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ParallelInteraction | O(N) | **[Entry Point]** Constructs a new ParallelInteraction object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the binary data in a ByteBuf to ensure it conforms to the ParallelInteraction structure. Does not create an object. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size in bytes that a serialized ParallelInteraction occupies in a buffer, starting from a given offset. |
| clone() | ParallelInteraction | O(N) | Creates a deep copy of the object and its contained data structures. Essential for safe state management. |

## Integration Patterns

### Standard Usage

The class is intended to be used by the network protocol layer. A handler receives a buffer, validates the message type, and uses the static `deserialize` method to hydrate the object for further processing by game logic.

```java
// Executed within a Netty channel handler
void processIncomingBuffer(ByteBuf buffer) {
    // Assume message type and offset are already determined
    int messageOffset = ...;

    // Validate before deserializing to prevent resource exhaustion
    ValidationResult result = ParallelInteraction.validateStructure(buffer, messageOffset);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid ParallelInteraction: " + result.error());
    }

    // Deserialize into a usable object
    ParallelInteraction interaction = ParallelInteraction.deserialize(buffer, messageOffset);

    // Pass the object to the game state manager
    gameState.handleInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Reuse:** Do not hold onto a deserialized ParallelInteraction instance across multiple game ticks or network events. Its data represents a point-in-time snapshot and should be processed immediately. If state must be preserved, use the `clone` method to create a safe copy.

-   **Concurrent Modification:** Never write to a ParallelInteraction instance from one thread while another thread might be reading from or serializing it. This will lead to race conditions and data corruption.

-   **Ignoring Validation:** Bypassing `validateStructure` on untrusted input exposes the system to malformed packets that could trigger uncaught exceptions or resource allocation attacks during deserialization.

## Data Pipeline

The ParallelInteraction class is a critical link in the chain that transforms raw network bytes into actionable game events.

> Flow:
> Raw ByteBuf from Netty Pipeline -> Protocol Frame Decoder -> **ParallelInteraction.deserialize** -> Game Event Handler -> Entity Component System -> Player State Update

