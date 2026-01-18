---
description: Architectural reference for Range
---

# Range

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Range {
```

## Architecture & Concepts
The Range class is a fundamental Plain Old Java Object (POJO) within the Hytale network protocol layer. It serves as a low-level data structure representing a numerical interval with a minimum and maximum value.

Architecturally, its primary role is to provide a standardized, fixed-size binary representation for a range of integers. The class is designed for high-performance serialization and deserialization directly to and from Netty ByteBufs. The presence of static constants like FIXED_BLOCK_SIZE and MAX_SIZE, both set to 8, underscores its design as a predictable, 8-byte data block (two 4-byte little-endian integers).

This class is not a service or manager; it is a building block used to compose more complex network packets. It embodies the protocol's strategy of using simple, blittable types to minimize serialization overhead.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1.  By the network deserialization pipeline, which invokes the static `deserialize` factory method when parsing an incoming data stream.
    2.  Manually by higher-level game or server logic that needs to construct a network packet for transmission (e.g., `new Range(10, 20)`).
- **Scope:** The lifetime of a Range object is typically ephemeral. It exists only within the scope of the network packet it belongs to or the immediate operation that created it. It is not intended for long-term storage.
- **Destruction:** As a standard Java object with no native resources, instances are managed by the Garbage Collector and are eligible for collection once all references are dropped.

## Internal State & Concurrency
- **State:** The state is **mutable**. The public fields *min* and *max* can be modified directly at any time after instantiation. This design prioritizes performance and ease of use over encapsulation.

- **Thread Safety:** This class is **not thread-safe**. Due to its mutable public fields, concurrent access from multiple threads without external synchronization will lead to race conditions and unpredictable behavior.

    **WARNING:** Never share a Range instance across threads without implementing proper locking mechanisms. It is strongly recommended to treat instances as thread-local or to create defensive copies using the clone method.

## API Surface
The public API is focused exclusively on serialization, validation, and standard object methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | Range | O(1) | **Static Factory.** Reads 8 bytes from the buffer at the given offset and constructs a new Range object. |
| serialize(ByteBuf) | void | O(1) | Writes the min and max values as two 4-byte little-endian integers into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant fixed size of the binary representation, which is always 8. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if the buffer contains at least 8 readable bytes from the given offset. Does not validate logical correctness. |
| clone() | Range | O(1) | Creates a shallow copy of the Range object. |

## Integration Patterns

### Standard Usage
The Range class is almost always used as a component of a larger, more complex protocol object. The parent object's serialization and deserialization logic is responsible for invoking the corresponding methods on the Range instance.

```java
// Example: Deserializing a Range from a network buffer
ByteBuf networkBuffer = ...;
int currentOffset = ...;

// Validate there is enough data for a Range
ValidationResult result = Range.validateStructure(networkBuffer, currentOffset);
if (!result.isOk()) {
    throw new ProtocolException(result.getErrorMessage());
}

// Deserialize the Range
Range damageRange = Range.deserialize(networkBuffer, currentOffset);
int bytesRead = Range.computeBytesConsumed(networkBuffer, currentOffset);
currentOffset += bytesRead;

// Use the deserialized object
processDamage(damageRange.min, damageRange.max);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Modification:** Do not modify a Range object after it has been passed to a system that may cache it or use it later. Its mutability can lead to hard-to-debug side effects. If a different range is needed, create a new instance.
- **Concurrent Sharing:** Never pass a reference to a Range object between threads without explicit synchronization. This is a direct path to data corruption.
- **Complex Logic:** Do not add business logic to this class. It is strictly a data container for the protocol layer.

## Data Pipeline
The Range class is a passive data container that flows through the network serialization and deserialization pipelines.

> **Outbound (Serialization) Flow:**
> Game Logic creates `Range` -> Parent Packet calls `range.serialize(buffer)` -> Netty `ByteBuf` -> Network Socket

> **Inbound (Deserialization) Flow:**
> Network Socket -> Netty `ByteBuf` -> Parent Packet calls `Range.deserialize(buffer, offset)` -> Game Logic receives `Range` instance

