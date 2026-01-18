---
description: Architectural reference for StringParamValue
---

# StringParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class StringParamValue extends ParamValue {
```

## Architecture & Concepts
The **StringParamValue** class is a fundamental component of the Hytale network protocol layer. It serves as a specialized Data Transfer Object (DTO) responsible for the serialization and deserialization of a single, nullable string. This class is not a high-level service; rather, it is a low-level building block used to compose more complex network packets.

Its primary architectural purpose is to encapsulate the precise binary format of a string on the wire. This includes handling:
-   **Nullability:** A leading bitmask indicates whether the string is present or null, conserving bandwidth for optional fields.
-   **Variable-Length Encoding:** It leverages a VarInt prefix to define the string's length, allowing for efficient representation of strings of varying sizes.
-   **Boundary and Safety Checks:** It enforces strict length limits (4,096,000 bytes) during deserialization to prevent buffer overflows and protocol-level Denial-of-Service attacks.

By abstracting these byte-level concerns, **StringParamValue** provides a clean, object-oriented interface to higher-level game logic and packet definitions, ensuring a consistent and safe data contract between the client and server.

### Lifecycle & Ownership
-   **Creation:** Instances are created under two scenarios:
    1.  **Inbound:** The static factory method **deserialize** is invoked by a parent packet's deserialization logic when parsing an incoming network buffer (a Netty **ByteBuf**).
    2.  **Outbound:** Higher-level application logic instantiates it directly via `new StringParamValue("...")` when constructing a packet to be sent.
-   **Scope:** The lifecycle of a **StringParamValue** instance is extremely brief and tied to the lifecycle of its containing packet. It is considered a transient, short-lived object.
-   **Destruction:** The object is eligible for garbage collection as soon as the parent packet has been fully processed (either read by game logic or written to the network socket). There are no manual resource management or cleanup methods.

## Internal State & Concurrency
-   **State:** The class holds a single, mutable public field: **value**. This design prioritizes performance and ease of access within the single-threaded context of packet processing, at the expense of encapsulation. The state is simple and contains no derived or cached data.
-   **Thread Safety:** **StringParamValue** is **not thread-safe**. Its public mutable state makes it susceptible to race conditions if shared across threads. It is designed exclusively for use within a single thread, typically a Netty I/O worker thread, which guarantees sequential processing of a given network stream.

**WARNING:** Never share an instance of **StringParamValue** between threads without explicit, external synchronization. Doing so will lead to unpredictable behavior and data corruption.

## API Surface
The public API is divided between static utility methods for buffer interaction and instance methods for data manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static StringParamValue | O(N) | Constructs a new instance by reading from a **ByteBuf**. Throws **ProtocolException** on malformed data. |
| serialize(buf) | int | O(N) | Writes the instance's state to a **ByteBuf** and returns the number of bytes written. |
| computeSize() | int | O(N) | Calculates the number of bytes the instance will occupy when serialized. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Quickly calculates the size of a serialized object in a buffer without full deserialization. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs pre-flight checks on a buffer to verify data integrity before deserialization. |
| clone() | StringParamValue | O(1) | Creates a shallow copy of the object. The internal string reference is shared. |

*N = length of the string*

## Integration Patterns

### Standard Usage
**StringParamValue** is almost never used in isolation. It is designed to be a field within a larger packet structure, which manages the overall serialization and deserialization flow.

```java
// Hypothetical packet using StringParamValue
public class PlayerLoginPacket extends Packet {
    private StringParamValue playerName;

    // Called by the protocol engine to deserialize the packet
    @Override
    public void deserialize(ByteBuf buf) {
        // The packet manages the buffer offset and calls the static method
        this.playerName = StringParamValue.deserialize(buf, buf.readerIndex());
        buf.skipBytes(StringParamValue.computeBytesConsumed(buf, buf.readerIndex()));
    }

    // Called by the protocol engine to serialize the packet
    @Override
    public void serialize(ByteBuf buf) {
        // The packet delegates serialization to the ParamValue instance
        this.playerName.serialize(buf);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse a **StringParamValue** instance across multiple logical packets. They are inexpensive to create, and reusing them can lead to state bleeding and unintended side effects. Always create a new instance for each new outbound packet.
-   **Manual Deserialization:** Do not manually read from the buffer and populate a `new StringParamValue()`. The static **deserialize** method contains critical validation and logic that would be bypassed.
-   **Ignoring Return Values:** The **serialize** method returns the number of bytes written. While often not needed, ignoring it can be problematic in complex buffer management scenarios.

## Data Pipeline
The class operates at a very low level of the data pipeline, acting as a translator between raw bytes and a typed object.

> **Inbound Flow:**
> Netty ByteBuf -> Parent Packet Deserializer -> **StringParamValue.deserialize** -> StringParamValue Instance -> Game Logic

> **Outbound Flow:**
> Game Logic -> new StringParamValue(...) -> Parent Packet Serializer -> **StringParamValue.serialize** -> Netty ByteBuf

