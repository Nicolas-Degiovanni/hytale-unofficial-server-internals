---
description: Architectural reference for FluidParticle
---

# FluidParticle

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class FluidParticle {
```

## Architecture & Concepts
The FluidParticle class is a specialized Data Transfer Object (DTO) designed for high-performance network serialization. It is not a service or manager, but rather a fundamental data structure representing a single particle within the Hytale protocol. Its primary role is to serve as a concrete, language-level representation of a binary data structure that is transmitted between the client and server.

The architecture is heavily optimized for wire-format efficiency. It employs a hybrid fixed-size and variable-size block layout. A leading bitmask, *nullBits*, is used to indicate the presence of optional fields, minimizing payload size when those fields are not transmitted. This design pattern is critical for reducing bandwidth in a real-time environment where thousands of such objects may be sent per second.

This class is a self-contained component of the protocol layer, providing static methods for deserialization, size computation, and structural validation directly from a Netty ByteBuf. This allows the network stack to operate on raw byte streams without the overhead of unnecessary object instantiation, particularly for validation and skipping over data.

## Lifecycle & Ownership
- **Creation:** An instance of FluidParticle is created in one of two scenarios:
    1. By the game logic when a new particle effect needs to be generated and sent over the network.
    2. By the protocol deserialization layer, specifically the static *deserialize* method, when an incoming network packet is being parsed.
- **Scope:** The object's lifetime is intentionally brief. It exists only for the duration of a single transaction: either to be serialized into a buffer or to be consumed by game logic after being deserialized from a buffer. It does not persist between game ticks or network packets.
- **Destruction:** Instances are managed by the Java Garbage Collector. There are no manual resource management or explicit destruction methods. Ownership is transient and typically confined to the scope of a single method.

## Internal State & Concurrency
- **State:** The internal state is fully mutable. All fields are public and can be modified directly after instantiation. The class acts as a simple data container and does not encapsulate or protect its state.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game update loop. Concurrent modification from multiple threads will result in race conditions and undefined behavior. All synchronization must be handled externally by the calling system.

## API Surface
The public contract is centered around serialization and deserialization operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FluidParticle() | Constructor | O(1) | Creates an empty instance with all fields null or zero. |
| FluidParticle(systemId, color, scale) | Constructor | O(1) | Creates a fully initialized instance. |
| deserialize(buf, offset) | static FluidParticle | O(N) | Constructs a new FluidParticle by reading from a ByteBuf at a given offset. N is the length of the variable-sized fields. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Reads a buffer to determine how many bytes a serialized FluidParticle occupies without full deserialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a low-cost check on a buffer to ensure it contains a structurally valid FluidParticle. Does not validate semantic content. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the protocol layer or game systems that directly interact with network packets. The typical flow involves creating an object, populating it, and passing it to a serializer.

```java
// Example: Creating and serializing a particle
FluidParticle particle = new FluidParticle("hytale:water_splash", new Color(0, 128, 255), 1.5f);

// Assume 'buffer' is an allocated Netty ByteBuf
particle.serialize(buffer);

// The buffer is now ready to be sent over the network.
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a FluidParticle instance across multiple network messages. Its mutable nature makes this practice highly error-prone. Always create a new instance for each logical particle.
- **Concurrent Modification:** Never share a FluidParticle instance between threads without explicit and robust locking. The class is not designed for concurrent access.
- **Ignoring Validation:** Do not deserialize data from an untrusted source without first calling *validateStructure*. Bypassing this step can expose the application to buffer overflows or denial-of-service attacks from malformed packets.

## Data Pipeline
FluidParticle is not a processor in a pipeline; it is the **data** that flows through it.

**Outbound Flow (Serialization):**
> Game Logic -> Creates **FluidParticle** Instance -> `serialize()` Method -> Netty ByteBuf -> Network Socket

**Inbound Flow (Deserialization):**
> Network Socket -> Netty ByteBuf -> `deserialize()` Method -> Creates **FluidParticle** Instance -> Game Logic

