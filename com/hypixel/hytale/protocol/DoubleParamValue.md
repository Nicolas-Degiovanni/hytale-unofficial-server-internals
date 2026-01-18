---
description: Architectural reference for DoubleParamValue
---

# DoubleParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class DoubleParamValue extends ParamValue {
```

## Architecture & Concepts
The DoubleParamValue class is a fundamental, low-level component of the Hytale network protocol layer. It serves as a concrete implementation for serializing and deserializing a standard 64-bit double-precision floating-point number.

Its primary architectural purpose is to provide a predictable, fixed-size data structure for network transmission. By defining a constant size of 8 bytes (via `FIXED_BLOCK_SIZE`), it allows the protocol's deserialization logic to perform highly efficient, non-allocating reads and buffer-slicing operations. The class interacts directly with Netty's ByteBuf, using little-endian byte order for cross-platform consistency.

This class is not intended for general-purpose use within the game engine; its scope is strictly limited to the serialization and deserialization of network messages. It is one of many `ParamValue` subclasses that form the building blocks of complex network packets.

## Lifecycle & Ownership
- **Creation:** Instances are created under two circumstances:
    1. By a higher-level protocol message deserializer, which calls the static `deserialize` factory method to construct an object from an incoming network buffer.
    2. By game logic or a message builder when constructing an outgoing network message, typically via `new DoubleParamValue(someDouble)`.
- **Scope:** Transient and short-lived. An instance's lifetime is tightly coupled to the lifecycle of its parent network message. It exists only for the duration of message processing.
- **Destruction:** The object is eligible for garbage collection as soon as the parent network message is no longer referenced. There is no manual cleanup or resource management required.

## Internal State & Concurrency
- **State:** Mutable. The core state is a single public `double value` field. This design prioritizes performance by avoiding method call overhead for direct value access.
- **Thread Safety:** **Not thread-safe.** Direct, unsynchronized access to the public `value` field makes this class unsafe for concurrent modification. This is an intentional design choice, as protocol processing is expected to occur within a single, dedicated thread per connection (e.g., a Netty event loop thread).

**WARNING:** Do not share instances of DoubleParamValue across threads. Do not read the value from one thread while another might be writing to it.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | DoubleParamValue | O(1) | **[Factory]** Reads 8 bytes from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | int | O(1) | Writes the internal double value to the buffer using little-endian byte order. Returns bytes written (always 8). |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 8. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a bounds check to ensure at least 8 bytes are readable from the given offset. |
| clone() | DoubleParamValue | O(1) | Creates a new instance with a copy of the value. |

## Integration Patterns

### Standard Usage
This class is almost never used directly by feature developers. It is invoked by the protocol's code-generated message classes.

**Serialization (Constructing a message):**
```java
// A hypothetical message class uses DoubleParamValue for a field
PlayerPositionMessage msg = new PlayerPositionMessage();
msg.setX(new DoubleParamValue(128.5));

// The message's serialize method would then call:
// doubleParamValue.serialize(buffer);
```

**Deserialization (Parsing a message):**
```java
// A message deserializer reads a field from a buffer
// The offset is calculated by the parent message parser
DoubleParamValue x = DoubleParamValue.deserialize(buffer, currentOffset);
double playerX = x.value;
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not modify a DoubleParamValue instance after it has been passed to a `serialize` method. The serialized data in the buffer will not be updated, leading to state desynchronization.
- **Manual Deserialization:** Avoid manually reading from the buffer. The static `deserialize` method is the correct and safe way to construct an instance from network data.
- **Cross-Thread Sharing:** Never pass an instance to another thread. If the value is needed elsewhere, read the primitive `double` and pass that instead.

## Data Pipeline
The DoubleParamValue acts as a simple data container during the serialization and deserialization pipeline.

> **Incoming Data Flow:**
> Netty ByteBuf -> Protocol Message Deserializer -> **DoubleParamValue.deserialize()** -> `double` value consumed by Game Logic

> **Outgoing Data Flow:**
> `double` value from Game Logic -> `new DoubleParamValue()` -> Protocol Message Serializer -> **instance.serialize()** -> Netty ByteBuf

