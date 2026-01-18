---
description: Architectural reference for ClampConfig
---

# ClampConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class ClampConfig {
```

## Architecture & Concepts
The ClampConfig class is a fundamental data structure within the Hytale network protocol layer. It is not a service or manager, but rather a Plain Old Java Object (POJO) designed to represent the configuration for clamping a numerical value. Its sole purpose is to define a minimum bound, a maximum bound, and a normalization flag for a related value in a larger data packet.

This class is engineered for high-performance, low-level network I/O, evidenced by its direct manipulation of Netty's ByteBuf. The serialization and deserialization methods operate with explicit offsets and use Little-Endian byte order (via `getFloatLE` and `writeFloatLE`), ensuring consistent binary representation across different machine architectures.

The presence of static constants like FIXED_BLOCK_SIZE and MAX_SIZE, along with static helper methods for validation and deserialization, strongly indicates that ClampConfig is a component within a highly structured, possibly auto-generated, serialization framework. It serves as a primitive building block for composing more complex protocol messages.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  By a higher-level protocol message's deserializer when parsing an incoming network packet.
    2.  Manually by game logic when constructing a new protocol message that requires clamping parameters before being sent over the network.
- **Scope:** The lifetime of a ClampConfig object is typically ephemeral. It is owned by its parent protocol message and exists only for the duration of processing that message. It does not persist beyond the scope of a single network operation.
- **Destruction:** The object is managed by the Java Garbage Collector. It is de-referenced and eligible for collection as soon as its parent protocol message is no longer in use. No manual memory management is required.

## Internal State & Concurrency
- **State:** The internal state of ClampConfig is **Mutable**. Its core data fields—min, max, and normalize—are public and can be modified after instantiation. While this provides flexibility, it is expected that instances are treated as immutable value objects after being assigned to a parent message.

- **Thread Safety:** This class is **not thread-safe**. Direct access to its public fields without external synchronization makes it vulnerable to race conditions and memory consistency errors if an instance is shared and modified across multiple threads.

    **Warning:** A ClampConfig instance must never be shared between threads. It is designed to be confined to the thread that is processing a specific network packet or constructing an outgoing one.

## API Surface
The public API is minimal, focusing exclusively on serialization, state, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ClampConfig | O(1) | Constructs a new ClampConfig by reading 9 bytes from the buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the 9-byte binary representation of the object into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains at least 9 readable bytes at the offset. |
| computeSize() | int | O(1) | Returns the fixed binary size of the object, which is always 9 bytes. |
| clone() | ClampConfig | O(1) | Creates a new instance with a copy of the current object's state. |

## Integration Patterns

### Standard Usage
ClampConfig is almost never used in isolation. It is deserialized as part of a larger packet structure or instantiated to configure a component before serialization.

```java
// Example: Configuring a value before sending it in a message
SomeMessage message = new SomeMessage();
ClampConfig clampRules = new ClampConfig(0.0f, 100.0f, true);

// The ClampConfig instance is assigned to the parent message
message.setValueClamp(clampRules);

// The parent message's serialize method will then call clampRules.serialize()
ByteBuf buffer = ...;
message.serialize(buffer);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Directly calling `deserialize` without a prior call to `validateStructure` is dangerous. This bypasses critical buffer size checks and can lead to an IndexOutOfBoundsException, potentially crashing a network thread.
- **State Mutation After Assignment:** Modifying a ClampConfig object after it has been assigned to a parent message can lead to undefined behavior, especially if that message is queued for network transmission. Treat the object as immutable once configured.
- **Cross-Thread Sharing:** As stated previously, sharing an instance between threads without proper locking mechanisms will lead to severe concurrency issues. Do not do this.

## Data Pipeline

The data flow for ClampConfig is simple and linear, acting as a data container during serialization and deserialization processes.

**Inbound (Receiving Data):**
> Flow:
> Network ByteBuf -> Parent Message Deserializer -> **ClampConfig.deserialize()** -> Instantiated ClampConfig within Parent Message

**Outbound (Sending Data):**
> Flow:
> Game Logic Instantiation -> Parent Message -> **ClampConfig.serialize()** -> Network ByteBuf

