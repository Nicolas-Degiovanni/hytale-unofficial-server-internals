---
description: Architectural reference for ModelTransform
---

# ModelTransform

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ModelTransform {
```

## Architecture & Concepts

ModelTransform is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. Its primary responsibility is to encapsulate the complete spatial state of an in-game model or entity—specifically its position, body orientation, and look orientation.

Architecturally, this class is designed for extreme performance and predictability in network communication. It achieves this through a **fixed-size binary layout**. Every ModelTransform instance serializes to exactly 49 bytes, regardless of which of its fields are populated. This design choice simplifies buffer allocation and parsing on both the client and server, eliminating the overhead and complexity associated with variable-length data structures.

The class employs a bitmask pattern for handling optional fields. The first byte of the serialized data is a `nullBits` field, where each bit flags the presence or absence of a subsequent data block (Position, Direction, etc.). If a field is null, its corresponding space in the buffer is zero-filled, maintaining the constant 49-byte stride. This is a common low-level optimization that balances data density with parsing simplicity.

## Lifecycle & Ownership

-   **Creation:** ModelTransform instances are created under two primary scenarios:
    1.  **Inbound:** The static `deserialize` method is called by a network packet decoder when parsing an incoming data stream. This constructs a new ModelTransform from the raw bytes.
    2.  **Outbound:** Game logic systems (e.g., Physics, AI, Player Input) instantiate a ModelTransform via its constructor to capture an entity's current state before serialization and transmission.

-   **Scope:** Instances are ephemeral and short-lived. They are designed to represent a snapshot of state for a single network packet or a single game-tick update. They are not intended to be held as long-term state containers.

-   **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as all references to it are dropped, which typically occurs after a network packet has been fully processed or sent. There are no manual cleanup requirements.

## Internal State & Concurrency

-   **State:** The class is fully **Mutable**. Its public fields can be directly accessed and modified after instantiation. It acts as a simple data holder with no internal logic for state management.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms.

    **Warning:** Sharing a ModelTransform instance across multiple threads without external synchronization is unsafe and will lead to data corruption and race conditions. It is designed to be confined to a single processing thread, such as a Netty event loop or the main game thread.

## API Surface

The public API is focused entirely on serialization, state access, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ModelTransform | O(1) | Constructs a new instance from a fixed 49-byte block within a ByteBuf. |
| serialize(buf) | void | O(1) | Writes the instance's state into the provided ByteBuf as a 49-byte block. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 49. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer contains enough readable bytes for a valid object. |
| clone() | ModelTransform | O(1) | Creates a deep copy of the instance and its contained objects. |

## Integration Patterns

### Standard Usage

ModelTransform is primarily used by higher-level protocol encoders and decoders.

**Deserializing from a network buffer:**
```java
// In a packet handler, after validating the buffer
ValidationResult result = ModelTransform.validateStructure(packetBuffer, offset);
if (result.isOk()) {
    ModelTransform transform = ModelTransform.deserialize(packetBuffer, offset);
    // Use the transform to update an entity's state
    entity.applyTransform(transform);
}
```

**Serializing for network transmission:**
```java
// In a game system, preparing an entity state update
Entity entity = getEntityById(123);
ModelTransform transform = new ModelTransform(
    entity.getPosition(),
    entity.getBodyOrientation(),
    entity.getLookOrientation()
);

// The packet encoder will then call serialize
transform.serialize(outputBuffer);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not cache and re-use a ModelTransform instance across multiple ticks or for different entities. This is a common source of bugs where stale data is accidentally transmitted. Always create a fresh instance for new state.
-   **Concurrent Modification:** Do not read a ModelTransform on a network thread while it is being written to by the main game thread. Pass immutable copies or use a proper queuing mechanism.
-   **Ignoring Validation:** Never call `deserialize` without first ensuring the buffer has at least 49 readable bytes from the given offset. Failing to do so will result in an IndexOutOfBoundsException and likely terminate the connection.

## Data Pipeline

ModelTransform is a data payload that flows through the network stack. It does not actively process data itself.

**Inbound Data Flow (Receiving):**
> Flow:
> Raw TCP Bytes -> Netty ByteBuf -> Protocol Packet Decoder -> **ModelTransform.deserialize** -> Game Logic (Entity State Update)

**Outbound Data Flow (Sending):**
> Flow:
> Game Logic (Entity State Snapshot) -> **new ModelTransform()** -> Protocol Packet Encoder -> **modelTransform.serialize** -> Netty ByteBuf -> Raw TCP Bytes

