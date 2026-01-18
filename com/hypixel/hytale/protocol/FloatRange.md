---
description: Architectural reference for FloatRange
---

# FloatRange

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class FloatRange {
```

## Architecture & Concepts
The FloatRange class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or a manager, but a simple, high-performance data structure representing a fixed-size block of data in a network packet. Its sole purpose is to define a continuous range between two floating-point values and to manage its own serialization and deserialization logic.

This class is designed to integrate directly with the Netty framework, operating on Netty ByteBuf objects. The serialization format is fixed at 8 bytes, consisting of two 4-byte little-endian floats. This rigid, predefined structure is critical for protocol stability and performance, eliminating the need for complex parsing or size lookups during network I/O.

The presence of static constants like FIXED_BLOCK_SIZE and static methods like deserialize and validateStructure indicates that FloatRange is a component within a larger, possibly auto-generated, protocol definition system. This system relies on each data structure to be self-describing in terms of its size and layout, enabling efficient composition of complex network messages from simple, fixed-size parts.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1. By the network deserialization pipeline, which calls the static `deserialize` factory method to construct an object from an incoming ByteBuf.
    2. By game logic or higher-level services that need to construct a network message for transmission.
- **Scope:** Short-lived and transient. A FloatRange object typically exists only for the duration of processing a single network event or constructing a single outbound packet. It is a temporary container for data in transit.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required. Ownership is transferred by creating new instances; objects are not typically reused.

## Internal State & Concurrency
- **State:** The class holds a mutable state via its public `inclusiveMin` and `inclusiveMax` fields. This design choice prioritizes performance by allowing direct field access, avoiding the overhead of getter and setter method calls. This is a common pattern in performance-critical DTOs.
- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it susceptible to race conditions if an instance is shared and modified concurrently by multiple threads. It is designed to be used within a single-threaded context, such as a Netty I/O worker thread, or be protected by external synchronization mechanisms if sharing is unavoidable.

## API Surface
The public API is focused exclusively on serialization, deserialization, and size computation for the network layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static FloatRange | O(1) | Constructs a new FloatRange by reading 8 bytes from the buffer at the given offset. Throws IndexOutOfBoundsException if the buffer is too small. |
| serialize(ByteBuf) | void | O(1) | Writes the internal state (two floats) as 8 bytes into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a non-destructive check to ensure at least 8 readable bytes exist at the given offset. **CRITICAL:** Always call this before deserialize. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 8. |

## Integration Patterns

### Standard Usage
FloatRange is used either to encode data for sending or to decode received data. It should never be held as long-term state.

**Serialization (Sending Data)**
```java
// Example: Defining a damage range for a weapon
FloatRange damageRange = new FloatRange(10.5f, 22.0f);

// Assume 'packetBuffer' is the ByteBuf for the outgoing packet
damageRange.serialize(packetBuffer);
```

**Deserialization (Receiving Data)**
```java
// Assume 'inboundBuffer' is a received ByteBuf and 'offset' is the read position
int dataOffset = ...;
ValidationResult result = FloatRange.validateStructure(inboundBuffer, dataOffset);

if (result.isOk()) {
    FloatRange receivedRange = FloatRange.deserialize(inboundBuffer, dataOffset);
    // Process the receivedRange...
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. Failing to do so can lead to an IndexOutOfBoundsException that crashes the network processing thread.
- **Concurrent Modification:** Do not share a FloatRange instance across threads. If one thread is reading the fields while another is writing to them, the behavior is undefined.
- **Incorrect Offset Management:** The `offset` parameter in `deserialize` and `validateStructure` is absolute. Incorrectly calculating this offset will lead to data corruption by reading the wrong memory segment from the buffer.

## Data Pipeline
FloatRange acts as a simple marshalling and unmarshalling component in the network data flow. It translates between the raw byte representation in a network buffer and the in-memory Java object representation.

> **Deserialization Flow:**
> Netty Channel -> ByteBuf -> Protocol Dispatcher -> **FloatRange.deserialize(buffer, offset)** -> Game Logic

> **Serialization Flow:**
> Game Logic -> **new FloatRange()** -> **instance.serialize(buffer)** -> Protocol Assembler -> Netty Channel

