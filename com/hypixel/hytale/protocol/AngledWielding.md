---
description: Architectural reference for AngledWielding
---

# AngledWielding

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class AngledWielding {
```

## Architecture & Concepts
AngledWielding is a low-level Data Transfer Object (DTO) that represents a fixed-size, 9-byte data block within the Hytale network protocol. Its primary role is to provide a structured, type-safe Java representation for raw byte data related to entity orientation and item wielding.

This class is a fundamental component of the protocol serialization and deserialization layer. It does not contain any game logic. Instead, it acts as a schema definition, mapping specific byte offsets within a Netty ByteBuf to meaningful fields: *angleRad*, *angleDistanceRad*, and *hasModifiers*.

The design is optimized for performance, using direct, little-endian reads and writes to and from byte buffers. The static constants, such as FIXED_BLOCK_SIZE and MAX_SIZE, define the rigid, unchangeable binary layout of this structure, ensuring protocol compatibility between the client and server.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1. **Deserialization:** The static deserialize method is called by a higher-level packet processor when parsing an incoming network buffer. A new AngledWielding object is allocated on the heap to hold the parsed data.
    2. **Serialization:** Game logic on the sending side (client or server) instantiates a new AngledWielding object and populates its fields before passing it to a serializer.

- **Scope:** The lifetime of an AngledWielding object is extremely short. It is scoped to the processing of a single network packet. It is created, read by the game logic, and then becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** The state is entirely mutable. All fields are public and can be modified directly after instantiation. The class is a simple container for three primitive values. It performs no caching and holds no references to other complex objects.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created and accessed by a single thread, typically a Netty I/O worker thread responsible for handling a specific client connection. Sharing an instance of AngledWielding across multiple threads without explicit, external synchronization will lead to race conditions and unpredictable behavior.

## API Surface
The public contract is focused on binary serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AngledWielding | O(1) | **Static Factory.** Reads 9 bytes from the buffer at the given offset and constructs a new instance. Does not perform bounds checking. |
| serialize(buf) | void | O(1) | Writes the instance's fields as 9 bytes into the provided buffer in little-endian format. |
| computeSize() | int | O(1) | Returns the constant size of the binary structure, which is always 9. |
| validateStructure(buffer, offset) | ValidationResult | O(1) | **Static Validator.** Checks if the buffer contains enough readable bytes (9) from the given offset. Does not validate content. |

## Integration Patterns

### Standard Usage
AngledWielding is intended to be used as a component within a larger, more complex packet structure. The parent packet is responsible for managing the buffer offsets and calling the static deserialize method.

```java
// A hypothetical parent packet deserializer
public void read(ByteBuf buffer) {
    // ... read other packet fields ...
    int currentOffset = ...;
    ValidationResult result = AngledWielding.validateStructure(buffer, currentOffset);
    if (!result.isOk()) {
        // Handle protocol error
        return;
    }
    this.wieldingData = AngledWielding.deserialize(buffer, currentOffset);
    // ... continue reading the rest of the packet ...
}
```

### Anti-Patterns (Do NOT do this)
- **Offset Mismanagement:** Passing an incorrect offset to deserialize will result in reading garbage data from the buffer, leading to severe and hard-to-debug game state corruption. Always validate the buffer size with validateStructure before deserializing.
- **Cross-Thread Sharing:** Do not pass an AngledWielding instance from a network thread to a game logic thread without creating a deep copy. Modifying the object from one thread while another is reading it is unsafe.
- **Reusing Instances:** Do not attempt to reuse AngledWielding instances for multiple packets. They are cheap to create and should be treated as disposable data containers for a single scope of work.

## Data Pipeline
This class sits at the lowest level of the protocol object model, directly interfacing with the raw byte stream.

> Flow (Inbound):
> Netty ByteBuf -> Parent Packet Deserializer -> **AngledWielding.deserialize()** -> Game Logic (Entity System)

> Flow (Outbound):
> Game Logic (Entity System) -> new **AngledWielding()** -> Parent Packet Serializer -> Netty ByteBuf

