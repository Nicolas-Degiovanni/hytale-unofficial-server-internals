---
description: Architectural reference for Edge
---

# Edge

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Edge {
```

## Architecture & Concepts
The Edge class is a fundamental data structure within the Hytale network protocol layer. It is not a service or a manager, but a passive, serializable object designed to represent the visual properties of a line or border, specifically its color and width.

Its primary role is to act as a well-defined contract for how "edge" data is structured and encoded in a byte stream. The class provides the logic for converting between its in-memory object representation and its fixed-size, 9-byte network representation. This fixed-size layout is critical for high-performance, zero-allocation parsing on the network layer, as it allows parsers to deterministically read or skip over the data without needing to parse complex headers or length prefixes.

The serialization format is custom and highly optimized:
1.  **Nullability Bitfield (1 byte):** A single byte at the start where the first bit indicates if the subsequent ColorAlpha field is present or null.
2.  **Color Data (4 bytes):** The serialized representation of the ColorAlpha object. If the color is null, this space is occupied by four zero bytes to maintain the fixed block size.
3.  **Width Data (4 bytes):** The width value, encoded as a little-endian float.

This structure is a common pattern in the Hytale protocol for encoding optional fields within a fixed-size block, avoiding the overhead of variable-length encoding.

### Lifecycle & Ownership
-   **Creation:** Instances are created in two primary scenarios:
    1.  By the protocol layer when a network packet is received, via the static `deserialize` factory method.
    2.  By game logic when constructing an outgoing network packet, typically using the `Edge(ColorAlpha, float)` constructor.
-   **Scope:** The object is transient. Its lifetime is typically very short, existing only for the duration of processing a single network packet or game state update.
-   **Destruction:** Edge objects are managed by the Java Garbage Collector. They are eligible for collection as soon as they are no longer referenced by the network or game logic that created them. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** The Edge class is a mutable container for its two fields, color and width. Its state can be modified after construction. The `clone` method provides a deep copy mechanism, which is important for safely creating modifiable copies of an existing Edge.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization. It is designed to be created, manipulated, and read within a single thread, such as a Netty I/O worker thread or the main game logic thread.

**WARNING:** Sharing and concurrently modifying an Edge instance between multiple threads will result in race conditions and undefined behavior. Any multi-threaded access must be protected by external synchronization.

## API Surface
The public API is focused on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Edge() | constructor | O(1) | Creates an uninitialized Edge object. |
| Edge(ColorAlpha, float) | constructor | O(1) | Creates a fully initialized Edge object. |
| deserialize(ByteBuf, int) | static Edge | O(1) | Constructs an Edge object by reading 9 bytes from the given buffer at the specified offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a 9-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 9. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure at least 9 bytes are readable from the buffer at the given offset. |

## Integration Patterns

### Standard Usage
The most common use case is deserializing an Edge from a network buffer as part of a larger packet-parsing operation. The static `deserialize` method is the primary entry point for this.

```java
// Assume 'packetBuffer' is a Netty ByteBuf received from the network
// and 'currentOffset' is the read position for the Edge data.

// 1. Validate the buffer has enough data before attempting to read.
ValidationResult result = Edge.validateStructure(packetBuffer, currentOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid buffer for Edge: " + result.getErrorMessage());
}

// 2. Deserialize the object from the buffer at the known offset.
Edge edgeData = Edge.deserialize(packetBuffer, currentOffset);

// 3. Advance the buffer's reader index or offset for the next component.
int bytesConsumed = Edge.computeBytesConsumed(packetBuffer, currentOffset); // Always 9
currentOffset += bytesConsumed;

// 4. Use the deserialized object in game logic.
applyEdgeStyle(edgeData);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Directly calling `deserialize` without first calling `validateStructure` can lead to an IndexOutOfBoundsException if the buffer is smaller than expected. This can crash the network processing pipeline.
-   **Incorrect Offset Management:** The `offset` parameter is critical. Passing an incorrect offset will result in reading corrupted data and deserializing an invalid object, leading to subtle and hard-to-debug visual or logical errors.
-   **Reusing Instances Across Threads:** Do not deserialize an Edge on a network thread and pass the same instance to multiple game logic threads for modification. This is a classic race condition. Use the `clone` method to create safe, distinct copies for each thread if necessary.

## Data Pipeline
The Edge class is a key component in the transformation of raw network bytes into structured, usable game data.

> Flow:
> Raw Byte Stream (Netty) -> Protocol Frame Decoder -> **Edge.deserialize** -> In-Memory Edge Object -> Game Logic (e.g., Rendering System)

