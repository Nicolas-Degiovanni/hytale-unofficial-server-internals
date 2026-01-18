---
description: Architectural reference for VarInt
---

# VarInt

**Package:** com.hypixel.hytale.math.data
**Type:** Utility

## Definition
```java
// Signature
public final class VarInt {
```

## Architecture & Concepts
The VarInt class is a low-level, high-performance serialization utility responsible for encoding and decoding primitive integers and longs into a variable-length format. Its primary purpose is to minimize the byte-footprint of numeric data, which is critical for optimizing network bandwidth and disk space in the Hytale engine.

This component sits at the lowest level of the data serialization stack, directly manipulating byte streams. It is frequently invoked by higher-level systems such as the network packet serializer and the world chunk storage manager.

The implementation is based on two core industry-standard concepts:

1.  **Variable-Length Quantity (VLQ):** An encoding scheme that uses a variable number of bytes to represent an integer. Each byte uses 7 bits for data and the most significant bit (MSB) as a continuation flag. If the MSB is 1, more bytes follow; if it is 0, this is the final byte of the sequence. This is highly efficient for the common case of small numeric values.

2.  **ZigZag Encoding:** A method for mapping signed integers to unsigned integers so that numbers with a small absolute value (e.g., -1, 1, -2) have small unsigned values. This allows them to be efficiently encoded by the VLQ mechanism. The VarInt class uses this for all *signed* read and write operations.

## Lifecycle & Ownership
- **Creation:** Not applicable. VarInt is a final class with a private constructor and exposes only static methods. It is never instantiated and carries no object state.
- **Scope:** The static methods of this class are globally accessible throughout the application's lifetime. It is a foundational utility with no lifecycle management.
- **Destruction:** Not applicable. As no instances are created, no cleanup or destruction is required.

## Internal State & Concurrency
- **State:** **Stateless**. The VarInt class holds no internal state. All methods are pure functions whose output depends solely on their input arguments.
- **Thread Safety:** **Inherently Thread-Safe**. As a stateless utility with no mutable fields, all methods can be safely called from any thread without requiring external synchronization or locks.

## API Surface
The API provides symmetric read and write operations for 32-bit integers and 64-bit longs, with variants for signed and unsigned values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| writeUnsignedVarInt(int, DataOutput) | void | O(1) | Encodes an unsigned 32-bit integer and writes it to a stream. Max 5 bytes. |
| writeSignedVarInt(int, DataOutput) | void | O(1) | Applies ZigZag encoding to a signed 32-bit integer and writes it to a stream. |
| readUnsignedVarInt(DataInput) | int | O(1) | Reads a variable-length sequence from a stream and decodes it into a 32-bit integer. |
| readSignedVarInt(DataInput) | int | O(1) | Reads a variable-length sequence and applies ZigZag decoding to restore the signed integer. |
| writeUnsignedVarLong(long, DataOutput) | void | O(1) | Encodes an unsigned 64-bit long and writes it to a stream. Max 10 bytes. |
| writeSignedVarLong(long, DataOutput) | void | O(1) | Applies ZigZag encoding to a signed 64-bit long and writes it to a stream. |
| readUnsignedVarLong(DataInput) | long | O(1) | Reads a variable-length sequence from a stream and decodes it into a 64-bit long. |
| readSignedVarLong(DataInput) | long | O(1) | Reads a variable-length sequence and applies ZigZag decoding to restore the signed long. |

## Integration Patterns

### Standard Usage
The VarInt class is used when serializing data to a stream, such as a network buffer or a file. The writer and reader must agree on the type and order of data.

```java
// Example: Writing and reading a packet length prefix
// Assume 'out' is a DataOutputStream and 'in' is a DataInputStream

// --- SENDER ---
int packetLength = 250;
// Write the length as a VarInt to save space
VarInt.writeUnsignedVarInt(packetLength, out);
// ... write packet data ...

// --- RECEIVER ---
// Read the length to know how many bytes to expect
int receivedLength = VarInt.readUnsignedVarInt(in);
byte[] data = new byte[receivedLength];
in.readFully(data);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate VarInt using reflection. The class is designed as a static utility and will throw an UnsupportedOperationException.
- **Mismatched Signedness:** Using an unsigned read method for a value written with a signed method (or vice-versa) will result in data corruption. ZigZag encoding and decoding must be applied symmetrically.
- **Stream Corruption:** Failure to handle IOException or reading past the end of a VarInt sequence can leave a stream in a corrupted state. It is critical that the reader and writer protocols are perfectly matched.

## Data Pipeline
VarInt acts as a transformation stage in a larger data serialization pipeline.

> **Write Pipeline:**
> Primitive `int` or `long` -> **VarInt (ZigZag Encoding if signed)** -> **VarInt (VLQ Encoding)** -> `DataOutput` Byte Stream

> **Read Pipeline:**
> `DataInput` Byte Stream -> **VarInt (VLQ Decoding)** -> **VarInt (ZigZag Decoding if signed)** -> Primitive `int` or `long`

