---
description: Architectural reference for ViewBobbing
---

# ViewBobbing

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ViewBobbing {
```

## Architecture & Concepts
The ViewBobbing class is a data transfer object (DTO) operating exclusively within the network protocol layer. Its sole purpose is to encapsulate and transport camera effect configurations, specifically for view bobbing and camera shake, between the server and client.

This class is not a service or manager; it is a pure data container designed for high-performance network I/O. The architecture is optimized for a custom binary serialization format, evident in the static `deserialize` and instance `serialize` methods. A key pattern employed is the use of a `nullBits` bitmask. This single byte at the start of the serialized block acts as a header, indicating which of the subsequent nullable fields are present in the data stream. This avoids the overhead of sending sentinel values or extra length prefixes for optional data, thereby conserving network bandwidth.

## Lifecycle & Ownership
- **Creation:** A ViewBobbing instance is created under two circumstances:
    1.  By the network protocol decoder when an incoming packet is being parsed, via the static `deserialize` method.
    2.  By game logic that needs to construct and send a new camera configuration to a remote endpoint.
- **Scope:** The object's lifetime is extremely short. It is scoped to the processing of a single network packet or a single game state update. It is not designed to be cached or persist long-term.
- **Destruction:** The object becomes eligible for garbage collection as soon as the network packet has been fully processed or the data has been applied to the relevant rendering systems. Ownership is never transferred to long-lived services.

## Internal State & Concurrency
- **State:** The class holds mutable state via its public `firstPerson` field. The state is a direct, and potentially null, reference to a CameraShakeConfig object. The structure is simple and intended for direct manipulation by the owning thread.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms and its public field allows for unsynchronized mutation. It is designed to be created, populated, and read within a single, well-defined thread context, such as a Netty I/O thread or the main game update thread.

    **WARNING:** Sharing a ViewBobbing instance across threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public API is focused entirely on binary serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ViewBobbing | O(N) | Constructs a ViewBobbing instance by reading from a ByteBuf at a given offset. N is the size of the nested data. |
| serialize(buf) | void | O(N) | Encodes the object's state into a binary format and writes it to the provided ByteBuf. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will consume when serialized. Does not perform serialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Verifies that the data in a buffer represents a structurally valid ViewBobbing object without full deserialization. |
| clone() | ViewBobbing | O(N) | Performs a deep copy of the object and its contained CameraShakeConfig. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the protocol layer. A typical interaction involves deserializing from a network buffer, reading the data, and then discarding the object.

```java
// In a network packet handler
// The buffer 'buf' and 'offset' are provided by the network framework

ValidationResult result = ViewBobbing.validateStructure(buf, offset);
if (!result.isValid()) {
    // Handle corrupt or invalid packet
    throw new ProtocolException(result.error());
}

ViewBobbing config = ViewBobbing.deserialize(buf, offset);

// Apply the configuration to the game's rendering engine
RenderingSystem.applyCameraEffects(config.firstPerson);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not deserialize into an existing ViewBobbing object or reuse an instance for a new outgoing packet. These objects are cheap to create and reusing them can lead to subtle bugs from leftover state.
- **Multi-threaded Access:** Do not pass a ViewBobbing instance to another thread without deep-cloning it first. The receiving system should own its own copy to prevent race conditions.
- **Manual Serialization:** Do not attempt to manually read or write the `firstPerson` field to a buffer. The binary format relies on the `nullBits` header, which is managed exclusively by the `serialize` and `deserialize` methods. Bypassing these methods will break protocol compatibility.

## Data Pipeline
ViewBobbing acts as a data record within the network-to-game-state pipeline. It does not process data itself; it *is* the data.

> Flow:
> Raw Network ByteBuf -> Protocol Framer -> **ViewBobbing.deserialize** -> ViewBobbing Instance -> Rendering Engine -> Visual Camera Effect

