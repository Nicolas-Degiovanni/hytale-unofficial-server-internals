---
description: Architectural reference for RepulsionConfig
---

# RepulsionConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class RepulsionConfig {
```

## Architecture & Concepts
The RepulsionConfig class is a specialized Data Transfer Object designed for high-performance network communication. It is not a service or a manager; it is a plain data structure that directly maps to a fixed-size binary layout in the Hytale network protocol.

Its primary role is to represent a set of three floating-point values that define repulsion physics parameters: a radius, a minimum force, and a maximum force. The class acts as a "struct" for the protocol layer, providing a type-safe, object-oriented view of a raw 12-byte block of memory.

The design prioritizes serialization and deserialization speed by exposing public fields and using static methods for protocol operations. This avoids the overhead of getters, setters, and instance-based validation, which is instead handled by higher-level protocol processors that operate directly on network buffers.

## Lifecycle & Ownership
- **Creation:** Instances are created under two specific conditions:
    1.  By a network protocol decoder calling the static `deserialize` method when an incoming packet is being parsed.
    2.  By game logic that needs to construct and send this configuration, typically by using the `RepulsionConfig(radius, minForce, maxForce)` constructor.
- **Scope:** Transient. An instance of RepulsionConfig is extremely short-lived. It exists only to transfer data between a raw network buffer and the game's internal state. It is not intended to be stored long-term.
- **Destruction:** The object is managed by the Java garbage collector and becomes eligible for collection as soon as it is no longer referenced. In typical use cases, this happens immediately after its data has been read or written.

## Internal State & Concurrency
- **State:** The state is fully mutable and exposed via public fields. This is a deliberate design choice for performance, allowing protocol codecs and game logic to read and write data directly without method call overhead. The object contains no other hidden state.
- **Thread Safety:** **This class is not thread-safe.** It is a simple data container and provides no internal synchronization. It is designed to be confined to a single thread, such as a Netty I/O worker thread or the main game logic thread.

**WARNING:** Sharing a RepulsionConfig instance across threads without external locking mechanisms will lead to race conditions and unpredictable behavior. Do not modify an instance after it has been submitted to a network serialization queue.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static RepulsionConfig | O(1) | Reads 12 bytes from the buffer at the given offset and constructs a new RepulsionConfig. Does not perform bounds checking. |
| serialize(ByteBuf) | void | O(1) | Writes the three float fields (12 bytes) to the provided buffer in little-endian format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer contains at least 12 readable bytes from the given offset. Does not validate content. |
| computeSize() | int | O(1) | Returns the constant size of the binary structure, which is always 12. |

## Integration Patterns

### Standard Usage
RepulsionConfig is used by protocol handlers to encode or decode network data. Game logic should create an instance, populate it, and pass it to a network writer.

```java
// Example: Sending a new repulsion configuration
// Assume 'networkChannel' is an existing network pipeline component.

RepulsionConfig config = new RepulsionConfig(10.0f, 0.5f, 5.0f);
networkChannel.send(config); // The channel's handler will call config.serialize()
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to RepulsionConfig objects in game state components. Copy the float values into your own data structures instead. This object is for transport only.
- **Cross-Thread Modification:** Never modify a RepulsionConfig instance from a different thread after it has been created or passed to another system.

    ```java
    // BAD: Modifying an object after queuing it for network transmission
    RepulsionConfig config = new RepulsionConfig(10.0f, 0.5f, 5.0f);
    network.queueForSend(config);
    config.radius = 20.0f; // This is a race condition. The sent data is now unpredictable.
    ```

## Data Pipeline
The class serves as a bridge between raw network bytes and structured game data.

> **Inbound Flow:**
> Network ByteBuf -> Protocol Decoder -> **RepulsionConfig.deserialize()** -> Game Event or State Update

> **Outbound Flow:**
> Game Logic -> **new RepulsionConfig()** -> Protocol Encoder -> **RepulsionConfig.serialize()** -> Network ByteBuf

