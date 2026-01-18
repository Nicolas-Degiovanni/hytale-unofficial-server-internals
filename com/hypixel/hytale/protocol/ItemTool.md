---
description: Architectural reference for ItemTool
---

# ItemTool

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ItemTool {
```

## Architecture & Concepts
The ItemTool class is a Data Transfer Object (DTO) that represents the properties of a tool-type item within the Hytale protocol. It is not a service or manager, but rather a structured data container designed for efficient network serialization and deserialization.

Its primary role is to serve as the in-memory representation of tool data as it is transmitted between the client and server. The class encapsulates the logic for converting its state to and from the Hytale binary wire format, interacting directly with Netty's ByteBuf. This makes it a fundamental building block of the network layer, used in any packet that needs to convey detailed information about a tool, such as its material specifications and interaction speed.

The serialization format is custom and highly optimized, using a combination of fixed-size blocks, a bitmask for nullable fields, and variable-length integers (VarInt) for array counts. This design minimizes network overhead.

### Lifecycle & Ownership
- **Creation:** ItemTool instances are created in two primary scenarios:
    1. By the network layer when a packet containing tool data is received. The static `deserialize` method is called to construct an instance from a raw ByteBuf.
    2. By game logic when a new tool is created or its properties are being prepared for network transmission.
- **Scope:** An instance of ItemTool is short-lived. Its lifetime is typically bound to the scope of the network packet it belongs to or the specific game logic operation that created it. It is a value object, not a persistent entity.
- **Destruction:** Instances are managed by the Java Garbage Collector. There are no manual cleanup or `close` methods required. They are eligible for collection as soon as they are no longer referenced.

## Internal State & Concurrency
- **State:** The class holds mutable state. Its public fields, `specs` and `speed`, can be directly modified after instantiation. The internal data, particularly the `specs` array, is a mutable collection of other DTOs.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms such as locks or atomic variables. All fields are public and mutable, making concurrent access from multiple threads inherently unsafe.

    **WARNING:** All operations on an ItemTool instance, including serialization and field modification, must be performed by a single thread. Typically, this will be a Netty I/O thread for deserialization or the main game thread for logic processing. Passing an instance between threads requires external synchronization or creating a defensive copy via the `clone` method.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a protocol-defined data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ItemTool | O(N) | Constructs an ItemTool instance from a ByteBuf. Throws ProtocolException on malformed data or buffer underflow. N is the number of specs. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. Throws ProtocolException if constraints are violated. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the number of bytes an ItemTool occupies in a buffer without performing a full deserialization. Critical for advancing buffer read pointers. |
| computeSize() | int | O(N) | Calculates the number of bytes this instance will occupy when serialized. Useful for pre-allocating buffers. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a structural validation of the data in a buffer without creating an object. Essential for protecting against malformed packets from untrusted sources. |
| clone() | ItemTool | O(N) | Creates a deep copy of the ItemTool instance and its contained ItemToolSpec array. |

## Integration Patterns

### Standard Usage
ItemTool is primarily used by higher-level packet handlers that decode and encode game state. The typical flow involves deserializing from a buffer, processing the data, and potentially serializing a new or modified instance back into a buffer.

```java
// Deserializing from a network buffer
// Assumes 'buffer' is a ByteBuf received from the network
ValidationResult result = ItemTool.validateStructure(buffer, offset);
if (!result.isValid()) {
    throw new IOException("Invalid ItemTool data: " + result.error());
}
ItemTool tool = ItemTool.deserialize(buffer, offset);
int bytesRead = ItemTool.computeBytesConsumed(buffer, offset);
// ... advance buffer reader index by bytesRead

// Creating and serializing for network transmission
ItemTool newTool = new ItemTool(new ItemToolSpec[]{...}, 1.5f);
newTool.serialize(outputBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Do not read or write to an ItemTool instance from multiple threads simultaneously. This will lead to race conditions, inconsistent state, and data corruption during serialization.
- **Ignoring Validation:** Never call `deserialize` on a buffer from an untrusted source without first calling `validateStructure`. Bypassing validation can lead to uncaught ProtocolExceptions, which may crash the processing thread if not handled correctly.
- **State Reuse:** Do not modify an ItemTool instance after it has been passed to a serialization method if the same instance is intended to be used elsewhere. Its mutable nature can cause unexpected side effects. Use the `clone` method to create an independent copy if modifications are needed.

## Data Pipeline
ItemTool acts as a serializer and deserializer within the network data pipeline. It translates between the raw byte stream and a structured, in-memory object representation.

> **Flow (Client -> Server):**
> Game Logic -> **new ItemTool()** -> PacketSerializer -> `itemTool.serialize(buf)` -> Netty Channel -> Network
>
> **Flow (Server -> Client):**
> Network -> Netty Channel -> ByteBuf -> PacketDeserializer -> `ItemTool.deserialize(buf)` -> **ItemTool instance** -> Game Logic

---

