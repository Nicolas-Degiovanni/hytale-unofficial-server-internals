---
description: Architectural reference for BuilderToolBrushOriginArg
---

# BuilderToolBrushOriginArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (DTO) / Transient

## Definition
```java
// Signature
public class BuilderToolBrushOriginArg {
```

## Architecture & Concepts
The BuilderToolBrushOriginArg class is a low-level Data Transfer Object within the Hytale network protocol layer. It is not a service or a manager, but rather a concrete, type-safe representation of a single field within a larger network packet. Its sole purpose is to encapsulate the serialization and deserialization logic for the *brush origin* setting used by the in-game builder tools.

This class is part of a highly optimized, likely auto-generated, serialization framework. The presence of static methods like deserialize, computeBytesConsumed, and validateStructure is characteristic of this pattern. This design allows the parent packet processor to read, validate, and calculate the size of its fields without needing to instantiate the containing object first, which is critical for performance and buffer management in a high-throughput networking environment like Netty.

The constants such as FIXED_BLOCK_SIZE and MAX_SIZE are metadata consumed by this framework to understand the binary layout of the object on the wire, confirming it is a fixed-size, single-byte structure.

## Lifecycle & Ownership
- **Creation:** Instances are created under two scenarios:
    1. **Inbound:** The static deserialize method is called by a parent packet's deserialization logic when decoding an incoming network buffer.
    2. **Outbound:** Game logic instantiates this object to configure a parent packet that is about to be sent to the server or client.
- **Scope:** The lifecycle of a BuilderToolBrushOriginArg instance is extremely brief and tied to a single network operation. It is a transient object that exists only for the duration of packet serialization or deserialization.
- **Destruction:** The object becomes eligible for garbage collection immediately after the parent packet has been fully processed or written to the network buffer. Ownership is never transferred; it is treated as a value type.

## Internal State & Concurrency
- **State:** The class holds a single mutable field, defaultValue, of type BrushOrigin. While technically mutable, it is designed to be set at creation and then treated as immutable. It contains no caching or complex state.
- **Thread Safety:** **This class is not thread-safe.** The public defaultValue field can be modified without any synchronization. This is by design, as packet processing is typically confined to a single Netty I/O worker thread.

    **WARNING:** Sharing an instance of BuilderToolBrushOriginArg across multiple threads is a severe design error and will lead to unpredictable behavior and data corruption. Instances MUST be confined to the thread that is processing the network packet.

## API Surface
The public API is designed for integration with a protocol serialization engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BuilderToolBrushOriginArg | O(1) | Constructs an instance by reading one byte from the buffer at the given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal BrushOrigin enum value as a single byte to the buffer. |
| computeSize() | int | O(1) | Returns the constant size of this object on the wire, which is 1 byte. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer is large enough to contain this object. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by high-level game systems. It is an implementation detail of a parent network packet. Its methods are invoked by the serialization and deserialization logic of a containing packet.

```java
// Hypothetical usage within a parent packet's serialization routine

// 1. An instance is created and attached to the parent packet
BuilderToolUpdatePacket packet = new BuilderToolUpdatePacket();
packet.brushOrigin = new BuilderToolBrushOriginArg(BrushOrigin.Corner);

// 2. The parent packet's serialize method delegates serialization
public void serialize(ByteBuf buf) {
    // ... serialize other packet fields
    this.brushOrigin.serialize(buf); // Correct usage
    // ... serialize remaining fields
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Deserialization:** Never manually read from a ByteBuf to populate a new instance. The static deserialize method is the designated factory and contract for the protocol.
    ```java
    // INCORRECT
    ByteBuf myBuffer = ...;
    BuilderToolBrushOriginArg arg = new BuilderToolBrushOriginArg();
    arg.defaultValue = BrushOrigin.fromValue(myBuffer.readByte());
    ```
- **Reusing Instances:** Do not reuse instances of this class across different packets. Its state is mutable and intended for a single serialization operation. Create a new instance for each new packet.
- **Ignoring Validation:** Bypassing the static validateStructure method before attempting to deserialize can lead to buffer underflow exceptions and server instability if presented with a malformed packet.

## Data Pipeline
This component acts as a translator between the raw byte stream and a type-safe object representation for a specific data field.

> **Inbound Flow (Client -> Server):**
> Raw TCP Bytes -> Netty ByteBuf -> Parent Packet Deserializer -> **BuilderToolBrushOriginArg.deserialize()** -> In-memory Packet Object -> Game Logic
>
> **Outbound Flow (Server -> Client):**
> Game Logic -> In-memory Packet Object -> **instance.serialize()** -> Netty ByteBuf -> Raw TCP Bytes

