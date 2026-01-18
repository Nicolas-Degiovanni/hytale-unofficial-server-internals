---
description: Architectural reference for ParamValue
---

# ParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Polymorphic Data Model

## Definition
```java
// Signature
public abstract class ParamValue {
```

## Architecture & Concepts
The ParamValue class is the abstract foundation for a polymorphic, type-length-value (TLV) serialization system within the Hytale network protocol. It serves as the bridge between raw binary data streams, managed by Netty's ByteBuf, and strongly-typed data structures used by the game engine.

This class and its concrete subclasses (StringParamValue, IntParamValue, etc.) are not intended for direct use in high-level game logic. Instead, they are a fundamental component of the protocol layer, enabling network packets to contain fields with variable data types.

The core architectural pattern is a static factory method, **deserialize**, which reads a prefixed type identifier from the byte stream. This identifier, a variable-length integer (VarInt), dictates which concrete subclass should be instantiated to parse the subsequent data. This design allows for protocol extensibility; new parameter types can be added without breaking the core deserialization logic.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the static `ParamValue.deserialize` factory method. This method is invoked by higher-level packet deserializers when they need to read a polymorphic value from an incoming ByteBuf. They are never instantiated directly by application code.

- **Scope:** Highly transient. A ParamValue object exists only for the duration of a single network packet's processing cycle. It is created, its value is read and transferred to a game-state object, and it immediately becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no explicit cleanup or `close` methods. Ownership is never transferred outside the scope of the packet processing handler.

## Internal State & Concurrency
- **State:** The ParamValue base class is stateless. All state, such as the actual string or integer value, is held within its concrete subclasses. Once deserialized from a buffer, these objects are effectively immutable Data Transfer Objects (DTOs).

- **Thread Safety:** **Not thread-safe.** All methods that interact with a Netty ByteBuf are stateful with respect to the buffer's reader and writer indices. These operations are designed to be executed synchronously within a single Netty I/O thread. Concurrent calls to `serialize` or `deserialize` on the same buffer will result in data corruption and unpredictable behavior. While a deserialized instance is safe to *read* from multiple threads, its intended lifecycle is confined to a single thread.

## API Surface
The public API is primarily for use by other protocol-layer components, such as packet serializers and deserializers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ParamValue | O(N) | Reads a type ID and data from the buffer to construct a concrete subclass. Throws ProtocolException on unknown type ID. |
| serializeWithTypeId(ByteBuf) | int | O(N) | Writes the type ID followed by the object's payload to the buffer. This is the primary serialization entry point. |
| computeSizeWithTypeId() | int | O(1) | Calculates the total number of bytes required to serialize the object, including its type ID. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the size of a serialized ParamValue in a buffer without creating an object. Used for skipping data. |
| getTypeId() | int | O(1) | Returns the integer identifier for the concrete subclass. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Validates the binary structure in the buffer without allocating a full object. Reports errors for malformed data. |

## Integration Patterns

### Standard Usage
ParamValue is almost always used inside the `serialize` and `deserialize` methods of a larger network packet. The calling code is responsible for managing the buffer's reader/writer indices.

```java
// Example: Deserializing a value within a Packet
public void deserialize(ByteBuf buf) {
    // The reader index before this operation
    int startOffset = buf.readerIndex();

    // Deserialize the value
    this.someValue = ParamValue.deserialize(buf, startOffset);

    // IMPORTANT: Advance the buffer's reader index
    int bytesConsumed = ParamValue.computeBytesConsumed(buf, startOffset);
    buf.readerIndex(startOffset + bytesConsumed);
}

// Example: Serializing a value within a Packet
public void serialize(ByteBuf buf) {
    // someValue is an instance of a ParamValue subclass
    if (this.someValue != null) {
        this.someValue.serializeWithTypeId(buf);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Serialization:** Do not call the abstract `serialize` method directly. Always use `serializeWithTypeId`. Omitting the type ID makes the data unreadable by the corresponding `deserialize` method.
- **Index Mismanagement:** Forgetting to advance the ByteBuf reader index after a `deserialize` or `computeBytesConsumed` call. This is a common source of errors, causing the deserializer to read the same data repeatedly or enter an infinite loop.
- **Direct Instantiation:** Do not use `new StringParamValue("foo")` in application logic that consumes network data. The `deserialize` factory is the sole entry point for creating these objects from a byte stream.

## Data Pipeline

The ParamValue class is a critical component in the serialization and deserialization data pipelines.

**Deserialization (Inbound)**
> Flow:
> Netty ByteBuf -> Packet Deserializer -> **ParamValue.deserialize()** -> Concrete ParamValue Instance -> Game Logic

**Serialization (Outbound)**
> Flow:
> Game Logic -> Concrete ParamValue Instance -> **instance.serializeWithTypeId()** -> Packet Serializer -> Netty ByteBuf

