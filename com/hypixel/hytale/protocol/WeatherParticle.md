---
description: Architectural reference for WeatherParticle
---

# WeatherParticle

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class WeatherParticle {
```

## Architecture & Concepts
The WeatherParticle class is a data contract object, also known as a Data Transfer Object (DTO), designed for high-performance network serialization. It represents the state of a single particle used in the game's weather simulation system, such as a raindrop or snowflake.

Its primary architectural role is to serve as an in-memory representation of a network message component. The class is not a service or manager; it is pure data. The design explicitly avoids reflection-based serialization in favor of direct, manual `ByteBuf` manipulation. This approach maximizes performance and minimizes memory allocation, which is critical for the high-throughput demands of the game's network protocol.

The serialization format is custom and highly optimized, featuring:
- A **nullability bitfield**: A single byte at the start of the serialized block indicates which of the nullable fields (systemId, color) are present, conserving network bandwidth.
- A **fixed-size data block**: Core, non-nullable, fixed-width fields are grouped at the beginning for fast, predictable reads.
- A **variable-size data block**: Fields like strings, which have a variable length, are appended after the fixed block.

This structure allows for efficient validation and partial deserialization if necessary.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1. **Server-Side:** The game logic engine instantiates and populates a WeatherParticle to describe a weather effect that needs to be sent to clients.
    2. **Client-Side:** The static factory method `deserialize` is invoked by the network protocol layer to construct an instance from an incoming network `ByteBuf`.
- **Scope:** A WeatherParticle is a short-lived, transient object. Its lifetime is typically confined to the scope of a single network packet's serialization or deserialization process. It is not intended to be stored as long-term state.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required. It becomes eligible for collection as soon as it is no longer referenced by the network or game logic code.

## Internal State & Concurrency
- **State:** The internal state is fully **mutable**. All fields are public and can be modified directly after instantiation. This design choice prioritizes performance and reduces object allocation overhead by allowing instances to be reused and reconfigured, a common pattern in performance-critical game loops.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be owned and operated on by a single thread at any given time, such as a network thread or the main game loop thread. Concurrent modification from multiple threads will lead to race conditions and undefined behavior. External synchronization must be provided by the caller if an instance must be shared across threads.

## API Surface
The public API is dominated by static methods that operate on Netty `ByteBuf` objects, forming the contract for network protocol handlers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | WeatherParticle | O(N) | **[Factory]** Constructs a WeatherParticle by reading from a ByteBuf at a specific offset. N is the length of the variable string data. Throws ProtocolException on data corruption. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. N is the length of the systemId string. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Used for pre-allocating buffer capacity. |
| computeBytesConsumed(buf, offset) | int | O(N) | Reads just enough of the buffer to determine the total size of a serialized WeatherParticle without fully deserializing it. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | Performs a lightweight check on the buffer to ensure a valid WeatherParticle could be read, without allocating a new object. Checks for buffer bounds and length sanity. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the protocol layer. The server creates an instance, populates it, and passes it to a serializer. The client receives a buffer and uses the static `deserialize` method to reconstruct the object.

```java
// Example: Deserializing from a network buffer on the client
// Assume 'packetBuffer' is an incoming ByteBuf from the network.
// The 'offset' is determined by the parent packet structure.

int initialOffset = ...;
ValidationResult result = WeatherParticle.validateStructure(packetBuffer, initialOffset);

if (result.isOk()) {
    WeatherParticle particle = WeatherParticle.deserialize(packetBuffer, initialOffset);
    // Pass the particle to the rendering or particle simulation system
    gameClient.getParticleSystem().spawn(particle);
} else {
    // Handle corrupted packet
    log.error("Invalid WeatherParticle data: " + result.getErrorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to WeatherParticle instances in long-lived collections or as part of the persistent game state. They are designed for transient network transfer only.
- **Cross-Thread Modification:** Never modify a WeatherParticle instance from one thread while another thread might be reading from or serializing it. This will cause data corruption.
- **Ignoring Offset:** The `deserialize` and `validateStructure` methods require a precise offset. Passing an incorrect offset will lead to `IndexOutOfBoundsException` or, worse, silent data corruption. The caller is responsible for managing the buffer's read position.

## Data Pipeline
The WeatherParticle serves as a data container that flows from the server's game logic to the client's rendering engine.

> **Server-Side Flow:**
> Game State Change -> **WeatherParticle (new)** -> Packet Serializer -> `serialize(buf)` -> Network Transport Layer

> **Client-Side Flow:**
> Network Transport Layer -> `ByteBuf` -> **WeatherParticle.deserialize(buf, offset)** -> Game Logic / Particle System -> Render Engine

