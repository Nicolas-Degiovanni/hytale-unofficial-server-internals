---
description: Architectural reference for ParticleCollision
---

# ParticleCollision

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public class ParticleCollision {
```

## Architecture & Concepts
The ParticleCollision class is a low-level Data Transfer Object (DTO) that defines the physical behavior of a particle upon colliding with a game world block. It is not a service or manager, but rather a fundamental data contract used within the Hytale network protocol.

Its primary role is to serve as a fixed-size, 3-byte schema for serializing and deserializing particle interaction rules. This class encapsulates three key properties: the type of block it interacts with, the action to take upon collision, and how the particle's rotation is affected.

Due to its direct mapping to a byte layout, this class is a critical component for ensuring protocol compatibility between the client and server. Any changes to its structure or serialization logic would constitute a breaking protocol change. It is designed for high-performance network I/O, avoiding complex object graphs or variable-length fields.

### Lifecycle & Ownership
- **Creation:** ParticleCollision instances are created in two primary scenarios:
    1. By the network protocol layer via the static `deserialize` factory method when decoding an incoming packet from a Netty ByteBuf.
    2. Manually by game logic, typically within the particle effects system, to define the behavior of a new particle effect before it is serialized and sent over the network.
- **Scope:** This object is designed to be short-lived and transient. Its lifetime is typically bound to the scope of a single network packet's processing cycle or the configuration phase of a particle effect. It does not persist across sessions or game states.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as the parent network packet or particle definition is no longer referenced. There is no manual cleanup or `close` method.

## Internal State & Concurrency
- **State:** The state of ParticleCollision is fully **mutable**. Its three core fields, `blockType`, `action`, and `particleRotationInfluence`, are public and can be modified directly after instantiation. The class acts as a simple data container.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization mechanisms.

    **WARNING:** Accessing or modifying a ParticleCollision instance from multiple threads without external synchronization will lead to race conditions and unpredictable behavior. It is intended for use within a single thread, such as a Netty event loop thread for deserialization or the main game thread for logic processing.

## API Surface
The public API is focused on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ParticleCollision() | constructor | O(1) | Creates a default instance. |
| ParticleCollision(type, action, influence) | constructor | O(1) | Creates a fully configured instance. |
| deserialize(ByteBuf, int) | static ParticleCollision | O(1) | Constructs an instance by reading 3 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the 3-byte state of the instance into a buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the structure (always 3). |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer has enough readable bytes for a valid structure. |
| clone() | ParticleCollision | O(1) | Creates a shallow copy of the instance. |

## Integration Patterns

### Standard Usage
The class is primarily used by the protocol layer for data marshalling. Game logic may construct it for configuration purposes.

```java
// Example: Defining a particle that expires on hitting any solid block
ParticleCollision collisionRule = new ParticleCollision(
    ParticleCollisionBlockType.Solid,
    ParticleCollisionAction.Expire,
    ParticleRotationInfluence.None
);

// The rule is then passed to a particle effect builder, which will
// eventually serialize it into a network buffer.
ByteBuf buffer = Unpooled.buffer();
collisionRule.serialize(buffer);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a single ParticleCollision instance across multiple particle definitions without resetting its state. Its mutability can lead to unintended side effects.
- **Cross-Thread Modification:** Do not modify an instance on the main game thread while a network thread is actively serializing it. This will cause data corruption.
- **Assuming Variable Size:** Do not attempt to handle this object as if it has a variable size. The entire implementation is built around the `FIXED_BLOCK_SIZE` of 3 bytes.

## Data Pipeline
ParticleCollision is a data component, not a processing stage. It represents a piece of data that flows through the network and game logic pipelines.

> **Ingress Flow (Receiving Data):**
> Netty ByteBuf -> Protocol Packet Decoder -> **ParticleCollision.deserialize()** -> Particle System Logic

> **Egress Flow (Sending Data):**
> Particle System Logic -> **new ParticleCollision(...)** -> Protocol Packet Encoder -> **instance.serialize()** -> Netty ByteBuf

