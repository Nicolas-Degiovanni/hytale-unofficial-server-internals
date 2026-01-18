---
description: Architectural reference for ParticleAnimationFrame
---

# ParticleAnimationFrame

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Struct (Transient)

## Definition
```java
// Signature
public class ParticleAnimationFrame {
```

## Architecture & Concepts
The ParticleAnimationFrame class is a pure Data Transfer Object (DTO) that represents a fixed-size data structure within the Hytale network protocol. It is not a managed service or component; rather, it is a fundamental building block for defining the visual properties of a single frame within a particle effect's lifecycle.

The core architectural principle of this class is **performance through a predictable, fixed-size memory layout**. The entire object serializes to and from a rigid 58-byte block. This design eliminates the overhead of dynamic size calculation and bounds checking during high-frequency network I/O, which is critical for synchronizing numerous visual effects between the client and server.

A key feature is the use of a single-byte bitmask, referred to internally as *nullBits*, as the first byte of the serialized block. This field efficiently encodes the presence or absence of its four nullable, object-based fields (frameIndex, scale, rotation, color), allowing the protocol to support optional properties without resorting to variable-length encoding schemes.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The static factory method *deserialize* is the designated entry point for creating an instance from an incoming network ByteBuf. This is typically invoked by a higher-level protocol decoder within the Netty pipeline.
    2.  **Game Logic:** The game engine creates instances programmatically via its constructor when defining a new particle effect that is destined for serialization and transmission.
- **Scope:** The lifecycle of a ParticleAnimationFrame is extremely brief and transient. It is intended to exist only for the duration of a single operation: either the processing of an incoming network packet or the construction of an outgoing one.
- **Destruction:** Instances are not managed by any container or service. They become eligible for garbage collection as soon as the reference to them goes out of scope, typically immediately after the parent network packet has been fully processed or serialized.

## Internal State & Concurrency
- **State:** The class is **fully mutable**, with public fields that can be directly modified after instantiation. It is a simple data container and does not cache any information or maintain any state beyond the properties it represents.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and read within a single thread, such as a Netty event loop thread. Concurrent access from multiple threads without external synchronization will result in race conditions and undefined behavior. Do not share instances across threads; if a copy is needed, use the *clone* method.

## API Surface
The public API is focused exclusively on serialization, deserialization, and validation for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ParticleAnimationFrame | O(1) | Constructs an instance by reading a fixed 58-byte block from a buffer. The primary factory method. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 58-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 58. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer has sufficient readable bytes for a full object. |
| clone() | ParticleAnimationFrame | O(1) | Creates a new instance with a copy of the current object's data. |

## Integration Patterns

### Standard Usage
The canonical use case is deserializing the object from a network buffer as part of a larger packet decoding process. Validation should always precede deserialization.

```java
// In a protocol decoder, where 'buffer' is an incoming ByteBuf and 'offset' is the read index.

if (ParticleAnimationFrame.validateStructure(buffer, offset).isOk()) {
    ParticleAnimationFrame frame = ParticleAnimationFrame.deserialize(buffer, offset);

    // Pass the transient frame object to the particle rendering system
    particleSystem.processNewFrame(frame);

    // The 'frame' object is now typically discarded.
} else {
    // Handle malformed packet
}
```

### Anti-Patterns (Do NOT do this)
- **Retaining Instances:** Do not store ParticleAnimationFrame instances in long-lived collections or as member variables in services. They are transient data carriers, not persistent state objects. If you must store the data, clone it or copy its values to a more appropriate data structure.
- **Manual Deserialization:** Do not use `new ParticleAnimationFrame()` and then manually read from a ByteBuf to populate its fields. This bypasses the critical *nullBits* logic and will lead to data corruption. Always use the static *deserialize* factory method.
- **Ignoring Validation:** Calling *deserialize* on a buffer without first ensuring it contains at least 58 bytes at the specified offset is unsafe and will result in an IndexOutOfBoundsException.

## Data Pipeline
ParticleAnimationFrame serves as a data record that flows through the network and rendering pipelines.

> **Inbound Flow (Server to Client):**
> Network ByteBuf -> Netty Protocol Decoder -> **ParticleAnimationFrame** (Instance) -> Particle System Logic -> Graphics Renderer

> **Outbound Flow (Client to Server or Authoring):**
> Game Logic / Effect Editor -> **ParticleAnimationFrame** (Instance) -> Netty Protocol Encoder -> Network ByteBuf

