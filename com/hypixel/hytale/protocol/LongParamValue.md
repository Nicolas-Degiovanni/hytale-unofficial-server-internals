---
description: Architectural reference for LongParamValue
---

# LongParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class LongParamValue extends ParamValue {
```

## Architecture & Concepts
The LongParamValue class is a fundamental data structure within the Hytale network protocol layer. It serves as a concrete implementation for serializing and deserializing a 64-bit, little-endian signed integer (a Java long). This class is not intended for general-purpose arithmetic but acts as a specialized Data Transfer Object (DTO) for network communication.

As a subclass of ParamValue, it is part of a larger, type-driven serialization framework. The protocol engine identifies parameter types within a packet's schema and dispatches serialization or deserialization tasks to the corresponding ParamValue implementation.

The design of LongParamValue prioritizes performance and memory efficiency. It operates directly on Netty's ByteBuf, a zero-copy-capable byte buffer, to minimize overhead. The inclusion of static factory methods like **deserialize** and **validateStructure** allows the protocol framework to parse and validate incoming byte streams without needing to instantiate an object first, which is critical for handling high-throughput network traffic and defending against malformed packets.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Inbound:** The protocol's deserialization engine instantiates LongParamValue via the static **deserialize** method when it parses a network packet containing an 8-byte integer field.
    2.  **Outbound:** Higher-level game logic or packet construction utilities create new instances to populate a packet's payload before it is sent over the network.

- **Scope:** Extremely short-lived. A LongParamValue object's lifetime is typically bound to the processing of a single network packet. It is created, its data is read or written, and it immediately becomes a candidate for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this class. Ownership is transient and is never transferred; the object is used and discarded within a narrow scope.

## Internal State & Concurrency
- **State:** Mutable. The core state is a single public field, **value**. While technically mutable, instances are almost always treated as immutable after their initial creation. Modifying the state of a LongParamValue after it has been serialized or before it is read can lead to data desynchronization.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, used, and discarded within a single thread, typically a Netty I/O worker thread or the main game loop. It contains no internal locks or synchronization primitives. Sharing instances across threads without external locking will result in undefined behavior and is strongly discouraged.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and size calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static LongParamValue | O(1) | Constructs a new LongParamValue by reading 8 bytes from the buffer at the given offset. |
| serialize(buf) | int | O(1) | Writes the internal 64-bit value to the buffer. Returns the number of bytes written (always 8). |
| computeSize() | int | O(1) | Returns the constant size of the data type, which is always 8 bytes. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a bounds check to ensure at least 8 readable bytes exist in the buffer. |
| clone() | LongParamValue | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
LongParamValue is almost never used directly by feature developers. It is an internal component of the protocol engine. The following demonstrates its role during packet processing.

```java
// SCENARIO: Building an outbound packet
ByteBuf packetBuffer = ...;
LongParamValue timestamp = new LongParamValue(System.currentTimeMillis());
timestamp.serialize(packetBuffer); // The value is written to the buffer

// SCENARIO: Parsing an inbound packet
ByteBuf receivedBuffer = ...;
int dataOffset = ...; // Determined by the packet parser

// Validate before reading to prevent buffer over-read errors
ValidationResult result = LongParamValue.validateStructure(receivedBuffer, dataOffset);
if (result.isOk()) {
    LongParamValue receivedTimestamp = LongParamValue.deserialize(receivedBuffer, dataOffset);
    long serverTime = receivedTimestamp.value;
    // ... process serverTime
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not reuse a LongParamValue instance for multiple distinct values or across different packets. The cost of instantiation is negligible, and reusing instances can introduce subtle state-related bugs.

    ```java
    // BAD: Reusing the same object can cause confusion and errors
    LongParamValue param = new LongParamValue(100);
    packet1.addParam(param);
    param.value = 200; // This modifies the value inside packet1!
    packet2.addParam(param);
    ```

- **Concurrent Modification:** Do not access or modify a LongParamValue from multiple threads. It must only be used within the context of the thread that owns the packet processing logic.

- **Manual Serialization:** Avoid writing the long value to a buffer manually. Always use the **serialize** method to ensure the byte order (Little Endian) and format remain consistent with the protocol specification.

## Data Pipeline
LongParamValue is a low-level component that acts as a bridge between raw bytes and the engine's in-memory representation of a long integer.

**Outbound (Serialization)**
> Flow:
> Game Logic (long) -> **new LongParamValue(long)** -> serialize(ByteBuf) -> Netty Channel

**Inbound (Deserialization)**
> Flow:
> Netty Channel -> ByteBuf -> **LongParamValue.deserialize(ByteBuf)** -> Game Logic (long)

