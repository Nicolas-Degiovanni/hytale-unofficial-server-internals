---
description: Architectural reference for RemoveEntityInteraction
---

# RemoveEntityInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class RemoveEntityInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The RemoveEntityInteraction class is a specialized Data Transfer Object (DTO) designed for network serialization. It represents a specific, stateful command within the game's interaction system: the removal of an entity from the world. This class does not contain logic itself; it is a pure data container that describes the parameters and conditions of the removal event.

Its primary architectural function is to serve as a contract between the client and server for this specific game action. The serialization format is highly optimized for network performance, employing a hybrid fixed-size and variable-size block layout.

-   **Fixed Block:** The first 40 bytes of the serialized data contain primitive fields and offsets. This allows for extremely fast, direct-memory access to core properties.
-   **Nullable Bit Field:** The very first byte is a bitmask indicating which of the nullable, complex objects (like effects, rules, or camera) are present in the payload. This avoids wasting bandwidth on null data.
-   **Variable Block:** Following the fixed block is a data region for variable-length objects. The fixed block contains integer offsets pointing to the start of each object's data within this region. This structure accommodates complex, nested data without requiring a rigid, wasteful packet size.

This class is a fundamental building block of the network protocol, translating high-level game concepts into a compact binary representation.

### Lifecycle & Ownership
-   **Creation:** An instance is created under two circumstances:
    1.  **Sending Peer (e.g., Server):** Game logic instantiates and populates the object when an entity removal interaction needs to be communicated to a client.
    2.  **Receiving Peer (e.g., Client):** The static `deserialize` method is invoked by the network protocol decoder when a corresponding packet is read from the network buffer.
-   **Scope:** The object's lifetime is exceptionally short and confined to a single processing scope. It is created, serialized (or deserialized), processed by a handler, and then immediately becomes eligible for garbage collection.
-   **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `close` methods. Given its transient nature, it is typically dereferenced and collected moments after its use.

## Internal State & Concurrency
-   **State:** The state of a RemoveEntityInteraction instance is entirely **mutable**. All public fields can be modified after construction. This is by design, allowing network or game logic to build the object piece by piece before serialization. The object is a self-contained snapshot of a single interaction and does not cache external data.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, populated, and processed by a single thread, typically a Netty IO thread or the main game logic thread.

**WARNING:** Concurrent modification of a RemoveEntityInteraction instance from multiple threads will lead to unpredictable behavior, data corruption, and race conditions. All access must be externally synchronized if multi-threaded use is unavoidable.

## API Surface
The public contract is dominated by static methods for deserialization and validation, and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static RemoveEntityInteraction | O(N) | Constructs a new instance by reading from the given ByteBuf at a specific offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf and returns the number of bytes written. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a security and integrity check on the raw buffer data without full deserialization. Critical for preventing crashes from malformed packets. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object directly from a buffer. Used by decoders to advance the buffer's read index. |
| clone() | RemoveEntityInteraction | O(N) | Creates a deep copy of the object and its nested data structures. |

## Integration Patterns

### Standard Usage
The class is almost exclusively handled by the network layer. After deserialization, the resulting object is typically passed to a central interaction handler or dispatched on an event bus to be processed by the entity management system.

```java
// Executed by a network packet handler
// The ByteBuf contains the raw packet data
RemoveEntityInteraction interaction = RemoveEntityInteraction.deserialize(buf, offset);

// Dispatch the interaction to the game logic thread for processing
gameContext.getInteractionService().handle(interaction);
```

### Anti-Patterns (Do NOT do this)
-   **Object Pooling/Reuse:** Do not attempt to pool or reuse instances of RemoveEntityInteraction. They are lightweight objects whose state is specific to a single event. Reusing an instance can cause data from a previous event to leak into a new one.
-   **Manual Deserialization:** Never attempt to read fields manually from the ByteBuf. The binary layout is complex and subject to change. Always use the provided `deserialize` method.
-   **Ignoring Validation:** On a publicly exposed server, failing to call `validateStructure` on incoming data before attempting to deserialize it is a significant security vulnerability that can lead to remote code execution or denial-of-service attacks.

## Data Pipeline
The flow of data for a received RemoveEntityInteraction is linear and unidirectional, moving from the network hardware to the core game logic.

> Flow:
> Network Socket -> Netty ByteBuf -> **RemoveEntityInteraction.deserialize()** -> Game Event Bus -> Entity System -> World State Update

