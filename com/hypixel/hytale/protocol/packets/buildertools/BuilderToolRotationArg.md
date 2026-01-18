---
description: Architectural reference for BuilderToolRotationArg
---

# BuilderToolRotationArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolRotationArg {
```

## Architecture & Concepts
The BuilderToolRotationArg class is a low-level Data Transfer Object (DTO) designed for network serialization. It is not a high-level service or manager; its sole responsibility is to represent a single, fixed-size rotation argument within the binary structure of a larger network packet.

This class acts as a serialization contract, defining the precise one-byte binary layout for a Rotation value transmitted between the client and server. Its static methods, such as deserialize and validateStructure, are integral components of the protocol layer's packet processing pipeline. They provide a structured and validated mechanism for converting raw byte streams into a meaningful game-level data type.

The design, featuring explicit size constants like FIXED_BLOCK_SIZE, prioritizes performance and predictable binary layouts, which is critical for a high-throughput networking environment.

### Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Inbound:** The static deserialize method is called by a parent packet's deserialization logic when parsing an incoming network ByteBuf.
    2.  **Outbound:** Game logic instantiates it directly via its constructor to build a command or packet that will be sent over the network.
- **Scope:** The lifecycle of a BuilderToolRotationArg instance is extremely brief and tied to the scope of a single packet processing operation. It is created, its data is used, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed exclusively by the Java Garbage Collector. There are no manual cleanup or resource release requirements.

## Internal State & Concurrency
- **State:** Mutable. The public field *defaultValue* can be modified after instantiation. This design choice likely favors object reuse or performance by avoiding the creation of new objects for simple state changes, but it introduces risks if not handled carefully.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, manipulated, and read within a single, well-defined thread context, such as a Netty I/O worker thread. Concurrent access to the *defaultValue* field from multiple threads will lead to race conditions and unpredictable behavior.

    **WARNING:** Do not share instances of this class across threads. If data must be passed to another thread, either create a new instance or pass the internal Rotation enum value directly.

## API Surface
The public API is focused entirely on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | BuilderToolRotationArg | O(1) | **Static Factory.** Reads one byte from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | void | O(1) | Writes the internal Rotation value as a single byte into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant binary size of this object, which is always 1 byte. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Validator.** Performs a pre-flight check to ensure the buffer contains enough data to deserialize the object. |
| clone() | BuilderToolRotationArg | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
This class is intended to be used by higher-level packet objects during their own serialization and deserialization routines.

```java
// Example: Deserializing from a network buffer
public void read(ByteBuf buffer, int offset) {
    ValidationResult result = BuilderToolRotationArg.validateStructure(buffer, offset);
    if (!result.isOk()) {
        // Handle protocol error
        return;
    }
    this.rotationArg = BuilderToolRotationArg.deserialize(buffer, offset);
    // ... continue reading other packet fields
}

// Example: Serializing to a network buffer
public void write(ByteBuf buffer) {
    if (this.rotationArg == null) {
        this.rotationArg = new BuilderToolRotationArg(Rotation.None);
    }
    this.rotationArg.serialize(buffer);
    // ... continue writing other packet fields
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Use:** Modifying the *defaultValue* field after the object has been serialized or its value has been consumed by game logic is a critical error. This can lead to state desynchronization between the sender and receiver.
- **Cross-Thread Sharing:** Never pass an instance of BuilderToolRotationArg to another thread. Its mutable nature makes it inherently unsafe for concurrent access.
- **Ignoring Validation:** Bypassing the static validateStructure method before calling deserialize can result in buffer underflow exceptions and server instability if presented with a malformed or truncated packet.

## Data Pipeline
BuilderToolRotationArg is a single component in the broader network data pipeline. It translates between the raw byte stream and a structured data type.

> **Inbound Flow:**
> Network ByteBuf -> Parent Packet Deserializer -> **BuilderToolRotationArg.deserialize()** -> Game Logic (Consumes Rotation)

> **Outbound Flow:**
> Game Logic (Creates Rotation) -> new **BuilderToolRotationArg()** -> Parent Packet Serializer -> **BuilderToolRotationArg.serialize()** -> Network ByteBuf

