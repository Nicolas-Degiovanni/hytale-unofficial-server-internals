---
description: Architectural reference for VarInt
---

# VarInt

**Package:** com.hypixel.hytale.protocol.io
**Type:** Utility

## Definition
```java
// Signature
public final class VarInt {
```

## Architecture & Concepts
The VarInt class is a stateless, low-level serialization utility that provides static methods for encoding and decoding variable-length integers. This encoding scheme is a foundational component of the Hytale network protocol, designed to minimize the network footprint of integer data.

It acts as a codec helper, directly manipulating Netty ByteBuf instances. Its primary application is in the serialization of packet headers, such as packet identifiers and data lengths, where small integer values are common. By using fewer bytes for smaller numbers, it achieves significant bandwidth savings over fixed-size integer types like a standard 32-bit int. This class is not intended for direct use in general game logic; it is a specialized tool for the network serialization layer.

## Lifecycle & Ownership
- **Creation:** As a static utility class with a private constructor, VarInt is never instantiated. Its methods are invoked statically.
- **Scope:** This class has no state and therefore no lifecycle. Its methods are pure functions whose lifetime is scoped to their immediate invocation during a serialization or deserialization operation.
- **Destruction:** Not applicable. There are no instances to manage or clean up.

## Internal State & Concurrency
- **State:** The VarInt class is completely stateless. It contains no member variables and all operations are performed exclusively on the arguments passed to its static methods.
- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. However, the ByteBuf objects it operates on are **not** thread-safe. Callers are responsible for ensuring that read and write operations on a given ByteBuf are properly synchronized and not accessed concurrently from multiple threads. Failure to do so will result in data corruption or runtime exceptions from the underlying Netty buffer.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| write(ByteBuf buf, int value) | void | O(1) | Encodes an integer into VarInt format and writes it to the buffer. Throws IllegalArgumentException for negative values. |
| read(ByteBuf buf) | int | O(1) | Reads a VarInt from the buffer, advancing the reader index. Throws ProtocolException if the VarInt is malformed or exceeds 5 bytes. |
| peek(ByteBuf buf, int index) | int | O(1) | Reads a VarInt from a specific index without modifying the buffer's reader index. Returns -1 if the VarInt is incomplete or malformed. |
| length(ByteBuf buf, int index) | int | O(1) | Calculates the byte length of the VarInt starting at a specific index. Returns -1 if the VarInt is incomplete or malformed. |
| size(int value) | int | O(1) | Calculates the number of bytes required to encode the given integer as a VarInt. |

## Integration Patterns

### Standard Usage
The primary use case is within custom Netty channel handlers or packet serialization logic to frame network messages.

```java
// Example: Writing a packet length prefix
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

ByteBuf buffer = Unpooled.buffer();
int messageLength = 256;

// Write the length as a VarInt
VarInt.write(buffer, messageLength);

// ... write message payload ...

// Example: Reading the length prefix
int decodedLength = VarInt.read(buffer);

if (messageLength == decodedLength) {
    // Proceed with reading the payload
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create an instance via reflection. The class is designed to be used statically.
- **Ignoring Buffer State:** Do not call read without first ensuring enough bytes are available in the buffer. This can lead to an IndexOutOfBoundsException or a ProtocolException if a partial VarInt is read. Always check readableBytes before invoking read.
- **Ignoring Error Codes:** The peek and length methods return -1 to indicate an incomplete or invalid VarInt at the specified position. Code that does not check for this return value may operate on invalid assumptions about packet structure.

## Data Pipeline
VarInt is a fundamental building block within the network data pipeline. It acts as a transformer, converting between the in-memory integer representation and the on-the-wire byte representation.

> **Serialization Flow:**
> Packet Object -> Packet Serializer -> **VarInt.write(value)** -> ByteBuf -> Netty Channel

> **Deserialization Flow:**
> Netty Channel -> ByteBuf -> **VarInt.read()** -> Packet Deserializer -> Packet Object

