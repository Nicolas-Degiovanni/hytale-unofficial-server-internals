---
description: Architectural reference for BuilderToolFloatArg
---

# BuilderToolFloatArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolFloatArg {
```

## Architecture & Concepts
The BuilderToolFloatArg class is a fixed-size Data Transfer Object (DTO) used within the Hytale network protocol. It serves as a data contract, defining the structure for a floating-point argument used by in-game builder tools. Its primary role is to facilitate the serialization and deserialization of tool parameters between the client and server.

This class is not a service or a manager; it is a passive data structure. Its design prioritizes performance and predictability in the network layer. The fixed block size of 12 bytes (3 x 4-byte floats) allows for extremely fast, zero-allocation reads and predictable buffer calculations, which is critical for the performance of the protocol decoders. It represents a schema for a constrained float value, complete with a default, minimum, and maximum bound.

## Lifecycle & Ownership
- **Creation:** Instances are typically created in one of two ways:
    1. By the network protocol layer via the static `deserialize` factory method when an incoming packet is being parsed.
    2. Manually by game logic via the constructor when defining the parameters for a new builder tool before serialization.
- **Scope:** Ephemeral. An instance of BuilderToolFloatArg exists only for the duration of processing the network packet that contains it or the UI interaction that defines it. It is not registered in any global context and is not intended for long-term storage.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after the parent packet or operation has been fully processed.

## Internal State & Concurrency
- **State:** BuilderToolFloatArg is a highly mutable data container. Its core state consists of three public float fields: `defaultValue`, `min`, and `max`. There is no internal logic, validation, or caching. The public visibility of its fields allows for direct, low-overhead manipulation but places the burden of maintaining valid state on the caller.

- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives. It is designed to be created, manipulated, and read within a single-threaded context, such as a Netty I/O thread or the main game thread.

    **Warning:** Sharing an instance of BuilderToolFloatArg across threads without external locking will lead to race conditions and unpredictable behavior. Do not store instances in shared collections that are accessed by multiple threads.

## API Surface
The public API is focused on serialization, deserialization, and size computation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | BuilderToolFloatArg | O(1) | **Static Factory.** Reads 12 bytes from the buffer at the given offset and constructs a new instance. Does not perform validation; use `validateStructure` first. |
| serialize(ByteBuf) | void | O(1) | Writes the three float fields (12 bytes) to the provided buffer in Little Endian format. |
| computeSize() | int | O(1) | Returns the constant size of the object when serialized, which is always 12. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Utility.** Checks if the buffer contains at least 12 readable bytes from the specified offset. This is a crucial pre-condition for calling `deserialize`. |

## Integration Patterns

### Standard Usage
The most common pattern involves validating a buffer segment and then deserializing the object. This is the standard procedure within network packet handlers.

```java
// Standard deserialization from a Netty ByteBuf
int offset = ...; // The starting position of the data in the buffer
ByteBuf networkBuffer = ...;

if (BuilderToolFloatArg.validateStructure(networkBuffer, offset).isOk()) {
    BuilderToolFloatArg toolArg = BuilderToolFloatArg.deserialize(networkBuffer, offset);

    // Game logic now uses the deserialized data
    float value = toolArg.defaultValue;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not deserialize data into a pre-existing BuilderToolFloatArg instance. The `deserialize` method is a static factory that creates a new object. Reusing instances across different packets or operations can introduce subtle state corruption bugs.

- **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. Doing so can result in an IndexOutOfBoundsException if the buffer is smaller than expected, potentially crashing the network processing pipeline.

- **Cross-Thread Sharing:** Do not pass an instance from a Netty I/O thread to a game logic thread without creating a defensive copy or ensuring a proper memory-safe handoff. Its mutable nature makes it unsafe for direct concurrent access.

## Data Pipeline
BuilderToolFloatArg acts as a deserialization target within the network data pipeline. It transforms a raw byte sequence into a structured, usable Java object.

> Flow:
> Raw Network ByteBuf -> Protocol Packet Decoder -> **BuilderToolFloatArg.validateStructure()** -> **BuilderToolFloatArg.deserialize()** -> Game Logic (Builder Tool System)

