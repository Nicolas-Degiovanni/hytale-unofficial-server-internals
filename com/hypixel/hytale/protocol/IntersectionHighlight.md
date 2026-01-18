---
description: Architectural reference for IntersectionHighlight
---

# IntersectionHighlight

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class IntersectionHighlight {
```

## Architecture & Concepts
The IntersectionHighlight class is a low-level data structure that defines the binary representation of visual highlighting information for in-game object intersections. It serves as a fundamental component of the client-server communication protocol, acting as a Plain Old Java Object (POJO) for serialization and deserialization tasks.

This class is not a service or a manager; it is a pure data container. Its design is optimized for network performance, featuring a fixed-size binary layout and explicit serialization methods that operate directly on Netty ByteBufs. The presence of static constants like FIXED_BLOCK_SIZE and MAX_SIZE indicates a rigid and predictable data contract, which is critical for a deterministic protocol. Its primary responsibility is to ensure that highlight data can be reliably encoded into a byte stream by a sender and decoded back into a structured object by a receiver.

## Lifecycle & Ownership
- **Creation:** An IntersectionHighlight instance is created under two primary conditions:
    1.  **Deserialization:** The static factory method `deserialize` is invoked by a network packet decoder when an incoming ByteBuf is being processed. This is the most common creation path on the receiving end (e.g., the client).
    2.  **Direct Instantiation:** Game logic on the sending end (e.g., the server) creates a new instance using the constructor, populates its fields, and prepares it for serialization.
- **Scope:** The object's lifetime is extremely short and transient. It typically exists only within the scope of a single network packet processing cycle or a single game state update. It is not intended to be cached or persist beyond its immediate use.
- **Destruction:** As a simple POJO, it is managed by the Java Garbage Collector. Once all references are dropped, typically after the relevant game logic has consumed its data, the object is eligible for collection. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The public fields `highlightThreshold` and `highlightColor` can be modified directly after instantiation. The class is designed as a simple data bucket with no internal logic for state management.
- **Thread Safety:** This class is **not thread-safe**. It contains no locks, volatile keywords, or other concurrency primitives. It is designed to be created, populated, and read within a single thread, such as a Netty I/O thread or the main game loop thread.

**WARNING:** Sharing an IntersectionHighlight instance across multiple threads without external synchronization will lead to memory consistency errors and unpredictable behavior.

## API Surface
The public API is focused entirely on data transport and manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | IntersectionHighlight | O(1) | **[Factory]** Constructs an object by reading a fixed 8-byte block from a buffer. |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into a fixed 8-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size (8) of the binary representation. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer has enough readable bytes for deserialization. |
| clone() | IntersectionHighlight | O(1) | Creates a deep copy of the object and its contained Color object. |

## Integration Patterns

### Standard Usage
The class is intended to be used by higher-level network protocol handlers. The sender populates the object and passes it to a serializer, while the receiver uses the static deserialize method to reconstruct it from a network buffer.

```java
// Example: Decoding an IntersectionHighlight from a network buffer
// This code would typically exist within a packet handler.

ByteBuf incomingPacket = ...;
int dataOffset = ...; // The starting position of the data in the buffer

ValidationResult result = IntersectionHighlight.validateStructure(incomingPacket, dataOffset);
if (result.isOk()) {
    IntersectionHighlight highlight = IntersectionHighlight.deserialize(incomingPacket, dataOffset);
    // Use the highlight object in game logic
    applyHighlightEffect(highlight.highlightColor, highlight.highlightThreshold);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Byte Manipulation:** Do not attempt to read or write the highlight data byte-by-byte from a buffer. Always use the provided `serialize` and `deserialize` methods to ensure correctness and forward compatibility with the protocol.
- **State Caching:** Do not hold references to IntersectionHighlight objects for longer than necessary. They are transient data carriers, not long-lived state containers. Caching them can lead to stale data and increased memory pressure.
- **Cross-Thread Modification:** Never modify an IntersectionHighlight instance from one thread while another thread is reading from or serializing it. This will cause data races.

## Data Pipeline
The class acts as a translation point between a raw byte stream and a structured in-memory object.

> Flow (Client-Side Reception):
> Netty Channel -> ByteBuf -> Protocol Decoder -> **IntersectionHighlight.deserialize** -> Game Logic -> Rendering Engine

