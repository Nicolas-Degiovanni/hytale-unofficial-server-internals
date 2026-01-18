---
description: Architectural reference for ModelDisplay
---

# ModelDisplay

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ModelDisplay {
```

## Architecture & Concepts
The ModelDisplay class is a specialized Data Transfer Object (DTO) designed for high-performance network communication within the Hytale protocol layer. Its primary function is to represent the state of a 3D model attached to an entity, including its position, rotation, scale, and attachment points.

The core architectural feature of this class is its custom binary serialization format, which is optimized for both size and processing speed. The format is a hybrid structure composed of two distinct parts:

1.  **Fixed-Size Block (45 bytes):** This initial block contains data that is always present and has a predictable size. It includes a one-byte bitmask (`nullBits`) to indicate which nullable fields are present, followed by the fixed-size `Vector3f` data for transforms. Crucially, it also contains integer offsets that act as pointers to the location of variable-sized data.

2.  **Variable-Size Block:** This block appears immediately after the fixed-size block and contains data of unpredictable length, such as the `node` and `attachTo` strings. This design avoids costly network overhead for optional or variable data while maintaining fast, direct memory access for fixed fields.

This class is fundamental to the game's rendering and entity systems, acting as the canonical data structure for transmitting model attachment information between the server and client.

### Lifecycle & Ownership
-   **Creation:** An instance of ModelDisplay is created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` method is invoked by a network packet decoder (e.g., a Netty channel handler) when parsing an incoming byte stream. This is the most common creation path on the client.
    2.  **Direct Instantiation:** Game logic on the server creates a new instance (e.g., `new ModelDisplay(...)`) to define an entity's appearance before serializing it into an outgoing packet.

-   **Scope:** The object's lifetime is intentionally transient. It typically exists only for the duration of a single packet's processing cycle. Once its data has been consumed by the relevant game system (like the Entity-Component-System or rendering engine), it is no longer referenced and becomes eligible for garbage collection.

-   **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. The class manages no native resources and requires no explicit destruction.

## Internal State & Concurrency
-   **State:** The class is **mutable**. All of its fields are public and can be modified directly after instantiation. This design choice prioritizes performance and ease of use within the single-threaded context of a network or game loop, where the object is often built incrementally before being serialized.

-   **Thread Safety:** ModelDisplay is **not thread-safe**. It contains no internal locking or synchronization primitives.

    **Warning:** Concurrent modification of a ModelDisplay instance from multiple threads will result in data corruption and undefined behavior. All operations on a given instance must be confined to a single thread, such as a Netty event loop thread or the main game thread.

## API Surface
The public API is centered around serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ModelDisplay | O(N) | Constructs a ModelDisplay object by parsing the binary format from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into its binary format and writes it to the provided ByteBuf. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a security-critical check on the raw buffer to validate offsets and lengths *before* full deserialization, preventing malformed packet exploits. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the total number of bytes occupied by a serialized ModelDisplay within a buffer, allowing decoders to advance their read position correctly. |
| computeSize() | int | O(N) | Calculates the total byte size required to serialize the current state of the object. Useful for pre-allocating buffers. |
| clone() | ModelDisplay | O(1) | Creates a shallow copy of the object. Vector3f objects are cloned, but strings are not. |

## Integration Patterns

### Standard Usage
The class is designed to be used by higher-level packet objects. The packet's own serialization logic delegates to the static methods of ModelDisplay.

```java
// Example: Deserializing from within a parent packet's decode method
public void decode(ByteBuf buffer) {
    // ... read other packet fields ...
    this.model = ModelDisplay.deserialize(buffer, buffer.readerIndex());
    buffer.readerIndex(buffer.readerIndex() + ModelDisplay.computeBytesConsumed(buffer, buffer.readerIndex()));
    // ... read remaining fields ...
}

// Example: Creating and serializing a new model display
ModelDisplay playerHat = new ModelDisplay(
    "models/hats/tophat.hgm",
    "attachment.head",
    new Vector3f(0, 1, 0),
    null, // No rotation
    new Vector3f(1, 1, 1)
);

playerHat.serialize(outgoingPacketBuffer);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Reuse:** Do not reuse a ModelDisplay instance across multiple packets or game ticks. They are cheap to create and should be treated as immutable once sent or received to prevent state corruption.
-   **Ignoring Validation:** On a server, never call `deserialize` on a buffer received from a client without first calling `validateStructure`. Bypassing this step exposes the server to Denial of Service (DoS) attacks via maliciously crafted packets with invalid lengths or offsets.
-   **Manual Deserialization:** Do not attempt to read the fields manually from a ByteBuf. The binary format is complex, involving bitmasks and relative offsets. Always use the provided `deserialize` method to ensure correctness.

## Data Pipeline
The ModelDisplay class is a key component in the transformation of raw network bytes into actionable game state.

> **Ingress Flow (Client):**
> Raw TCP Bytes -> Netty ByteBuf -> Packet Decoder -> **ModelDisplay.deserialize** -> In-Memory ModelDisplay Object -> Entity Component System -> Renderer

The binary layout within the ByteBuf is strictly defined:

> **Binary Structure:**
> `[1-byte nullBitmask]` `[12-bytes translation]` `[12-bytes rotation]` `[12-bytes scale]` `[4-byte node_offset]` `[4-byte attachTo_offset]` `[variable-length string data...]`

