---
description: Architectural reference for InteractionSettings
---

# InteractionSettings

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class InteractionSettings {
```

## Architecture & Concepts
The InteractionSettings class is a low-level data structure that represents a specific configuration field within the Hytale network protocol. It is not a high-level service or manager, but rather a fundamental building block for network communication, analogous to a single field in a database schema.

Its primary role is to encapsulate the boolean state for `allowSkipOnClick`, ensuring a consistent and predictable binary representation on the wire. The class design, with its static serialization and validation methods, strongly indicates it is part of a larger, performance-critical protocol framework, likely driven by a code generator. This framework uses the static metadata fields like FIXED_BLOCK_SIZE to perform highly optimized, reflection-free serialization and deserialization directly from Netty ByteBuf objects.

This component lives exclusively within the network protocol layer and serves as a data contract between the client and server.

### Lifecycle & Ownership
- **Creation:** An InteractionSettings object is instantiated under two primary circumstances:
    1. By the protocol layer's deserialization logic when an incoming network packet is being parsed. The static `deserialize` factory method is the designated entry point.
    2. By game logic when constructing an outgoing packet that requires this specific settings block.

- **Scope:** The object's lifetime is extremely short and tied to the scope of a single network message processing task. It is created, read, and then becomes eligible for garbage collection. It is not intended to be cached or persist beyond the immediate transaction.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** The internal state is mutable, consisting of a single public boolean field, `allowSkipOnClick`. While technically mutable, instances should be treated as immutable after deserialization or before serialization to prevent data inconsistencies.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms.

    **WARNING:** All operations on an InteractionSettings instance must be confined to a single thread, typically a Netty I/O worker thread for deserialization or the main game thread for serialization. Sharing instances across threads without explicit external locking will lead to race conditions and undefined behavior.

## API Surface
The public API is designed for integration with a protocol codec, not for general-purpose application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static InteractionSettings | O(1) | Factory method. Constructs an instance by reading a fixed block of 1 byte from the provided buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal boolean state as a single byte into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object in bytes, which is always 1. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure the buffer contains enough readable bytes for a successful deserialization. |

## Integration Patterns

### Standard Usage
This object is almost never used directly by feature developers. It is managed by the underlying network packet system. The typical interaction is indirect, where a parent packet object is deserialized, which in turn deserializes this component.

```java
// Hypothetical usage within a packet's deserialization logic
// A parent packet object would invoke this during its own parsing routine.

public void deserialize(ByteBuf buffer, int offset) {
    // ... deserialize other parent packet fields ...
    this.interactionSettings = InteractionSettings.deserialize(buffer, someCalculatedOffset);
    // ... continue deserializing ...
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not deserialize into an existing InteractionSettings object or reuse a single instance for multiple outgoing messages. This can lead to state corruption. Always create a new instance for each distinct message.
- **Manual Byte Manipulation:** Do not attempt to read or write the `allowSkipOnClick` boolean to a ByteBuf manually. The `serialize` and `deserialize` methods guarantee correctness according to the protocol specification. Bypassing them risks protocol desynchronization.

## Data Pipeline
The InteractionSettings object is a transient data carrier in the network pipeline. It exists to convert structured data into a byte stream and back.

> Flow:
> Incoming Network ByteBuf -> Protocol Codec calls **InteractionSettings.deserialize** -> Instance passed to Game Logic -> (Optional) New instance created by Game Logic -> Protocol Codec calls **InteractionSettings.serialize** -> Outgoing Network ByteBuf

