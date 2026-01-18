---
description: Architectural reference for CooldownConditionInteraction
---

# CooldownConditionInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class CooldownConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The CooldownConditionInteraction class is a specialized Data Transfer Object (DTO) within the Hytale network protocol layer. It represents a node in a server-defined interaction graph where the player's ability to proceed is gated by a named cooldown timer. This class does not implement the cooldown logic itself; rather, it carries the *configuration* for an interaction that the client-side or server-side game logic will use to enforce such a condition.

Architecturally, this class is a concrete implementation of the more abstract SimpleInteraction. Its design is optimized for high-performance network transmission, prioritizing minimal memory allocation and low serialization/deserialization overhead. This is achieved through a custom binary format that operates directly on Netty ByteBufs, bypassing slower, more generic serialization frameworks.

The binary layout is a key design feature. It consists of two main parts:
1.  A **fixed-size block** containing primitive types and value types.
2.  A **variable-size block** for complex, nullable, or variable-length data structures like strings, maps, and arrays.

A single byte, the *nullable bit field*, at the beginning of the structure acts as a bitmask to efficiently declare which of the optional, variable-size fields are present in the payload. This avoids the need for terminator bytes or explicit null markers for every field, significantly compacting the data on the wire. Offsets to the variable data are stored in the fixed-size block, allowing for direct memory jumps during deserialization.

## Lifecycle & Ownership
-   **Creation:** Instances are created under two distinct circumstances:
    1.  **Deserialization:** The primary creation path. The static `deserialize` factory method is invoked by a network channel handler when a corresponding packet is read from a ByteBuf.
    2.  **Server-Side Definition:** Game logic on the server may instantiate and populate this object when defining an interaction graph that will later be serialized and sent to clients.

-   **Scope:** The object's lifetime is ephemeral. It is designed to exist only for the duration of processing a single network packet or a discrete game logic event. It is not a long-lived entity and is not intended to be cached or persisted.

-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are released, which typically occurs at the end of the network event or game tick in which it was created.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its fields are populated during construction or deserialization and are publicly accessible. It is effectively a struct passed between layers of the application.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It is designed to be confined to a single thread, such as a Netty I/O worker thread or the main game loop thread. Concurrent modification from multiple threads will result in data corruption and undefined behavior. Any multi-threaded access must be managed with external synchronization.

## API Surface
The public API is dominated by static methods for binary protocol handling rather than instance methods for business logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CooldownConditionInteraction | O(N) | **[Factory]** Constructs an instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a structural integrity check on the binary data in a buffer without full deserialization. Critical for security and stability. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size in bytes that the object occupies in the buffer. Used for advancing buffer read pointers. |
| clone() | CooldownConditionInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The class is almost exclusively used by the protocol decoding layer. A network handler receives a buffer, validates its structure, and then deserializes it into an object to be passed to the game engine.

```java
// Executed within a Netty channel handler or protocol decoder
void processInteractionPacket(ByteBuf packetData) {
    int offset = ... // Determine start of the object in the buffer

    // CRITICAL: Always validate before deserializing untrusted data.
    ValidationResult result = CooldownConditionInteraction.validateStructure(packetData, offset);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid CooldownConditionInteraction: " + result.error());
    }

    CooldownConditionInteraction interaction = CooldownConditionInteraction.deserialize(packetData, offset);

    // Pass the hydrated DTO to the game's interaction system.
    gameState.getInteractionManager().handleInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from a remote client without first calling `validateStructure`. Doing so exposes the server to malformed packets that can trigger exceptions, cause buffer over-reads, or lead to denial-of-service vulnerabilities.
-   **Long-Term Storage:** Do not hold references to this object beyond the immediate scope of its use. It is not designed for caching or long-term state management. Re-deserializing is the expected pattern.
-   **Concurrent Modification:** Do not share an instance of this class between threads. If data must be passed to another thread, create a deep copy using the `clone` method.

## Data Pipeline
This class serves as a data payload that is hydrated from a raw byte stream and passed to higher-level game systems. It is a critical link between the network transport layer and the game simulation layer.

> Flow:
> Raw ByteBuf from Network -> Protocol Dispatcher -> **CooldownConditionInteraction.validateStructure** -> **CooldownConditionInteraction.deserialize** -> Hydrated DTO Instance -> Game Interaction System -> Cooldown Manager Logic

---

