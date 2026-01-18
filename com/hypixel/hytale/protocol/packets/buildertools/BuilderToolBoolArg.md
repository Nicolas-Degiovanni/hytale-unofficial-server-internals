---
description: Architectural reference for BuilderToolBoolArg
---

# BuilderToolBoolArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BuilderToolBoolArg {
```

## Architecture & Concepts
The BuilderToolBoolArg class is a low-level data structure that represents a single boolean argument for a builder tool command within the Hytale network protocol. It is not a complete network packet itself, but rather a fundamental, serializable component used to compose more complex packets.

Its primary architectural role is to encapsulate the logic for serializing and deserializing a boolean value to and from a Netty ByteBuf. The class follows a self-contained serialization pattern, where the static methods for serialization, deserialization, and validation reside directly within the data class. This design avoids the need for separate serializer or codec classes for simple data types, promoting locality of reference and simplifying the protocol implementation.

This component is a primitive building block for the server-client communication related to in-game building and editing tools.

## Lifecycle & Ownership
- **Creation:** An instance is created in one of two scenarios:
    1.  By game logic when constructing an outbound packet that requires a boolean argument.
    2.  By the static deserialize method when a corresponding inbound packet is being parsed from a raw network buffer.
- **Scope:** The object's lifetime is extremely short and is strictly tied to its containing network packet. It exists only for the duration of packet construction and serialization, or for the duration of deserialization and processing.
- **Destruction:** The object is eligible for garbage collection as soon as the parent packet is processed or sent, and all references to it are dropped. There is no manual memory management.

## Internal State & Concurrency
- **State:** The class holds a single mutable public field, *defaultValue*. The state is minimal, representing only the boolean value it carries. The constants like FIXED_BLOCK_SIZE define its binary footprint.
- **Thread Safety:** This class is **not thread-safe**. The public, non-final field *defaultValue* can be modified without synchronization. This is considered acceptable because instances are intended to be confined to a single thread during their lifecycle—typically a Netty I/O thread for deserialization or a game logic thread for construction.

**WARNING:** Do not share instances of BuilderToolBoolArg between threads. Access must be confined to the thread that creates or deserializes the object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolBoolArg | O(1) | **Static.** Reads 1 byte from the buffer at the given offset and constructs a new instance. |
| serialize(buf) | void | O(1) | Writes the internal boolean state as a single byte (1 for true, 0 for false) into the buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 1 byte. |
| validateStructure(buffer, offset) | ValidationResult | O(1) | **Static.** Checks if the buffer contains enough readable bytes (at least 1) to deserialize the object. |
| clone() | BuilderToolBoolArg | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly by high-level game systems. It is exclusively used by the serialization and deserialization logic of a parent network packet.

```java
// Hypothetical parent packet's serialization logic
public void serialize(ByteBuf buf) {
    // ... other fields ...
    BuilderToolBoolArg enablePhysicsArg = new BuilderToolBoolArg(true);
    enablePhysicsArg.serialize(buf);
    // ... other fields ...
}

// Hypothetical parent packet's deserialization logic
public static MyPacket deserialize(ByteBuf buf, int offset) {
    // ...
    BuilderToolBoolArg enablePhysicsArg = BuilderToolBoolArg.deserialize(buf, currentOffset);
    int bytesConsumed = BuilderToolBoolArg.computeBytesConsumed(buf, currentOffset);
    currentOffset += bytesConsumed;
    // ...
    return new MyPacket(enablePhysicsArg.defaultValue);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the public *defaultValue* field after the object has been passed to a parent packet's constructor or serialize method. The serialized data will not reflect the change, leading to desynchronization.
- **Usage Outside Protocol Layer:** Do not use this class to pass boolean data within the game logic. Use the primitive boolean type. This class is strictly for network serialization.
- **Ignoring Validation:** Failing to call validateStructure before attempting to deserialize can lead to buffer under-read exceptions and server instability if presented with malformed or truncated packets.

## Data Pipeline

The BuilderToolBoolArg is a component within a larger data flow. It does not operate in isolation.

> **Outbound (Serialization) Flow:**
> Game Command -> Parent Packet Constructor(`new BuilderToolBoolArg(true)`) -> ParentPacket.serialize() -> **BuilderToolBoolArg.serialize(ByteBuf)** -> Netty Channel Pipeline

> **Inbound (Deserialization) Flow:**
> Netty Channel Pipeline -> ParentPacket.deserialize() -> **BuilderToolBoolArg.deserialize(ByteBuf, offset)** -> Parent Packet Instance -> Game Command Handler

