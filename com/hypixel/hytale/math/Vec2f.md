---
description: Architectural reference for Vec2f
---

# Vec2f

**Package:** com.hypixel.hytale.math
**Type:** Value Object / Transient

## Definition
```java
// Signature
public final class Vec2f {
```

## Architecture & Concepts
Vec2f is a fundamental value object representing a two-dimensional vector or coordinate using single-precision floating-point numbers. Its primary architectural role is to serve as a high-performance Data Transfer Object (DTO) for geometry, physics, and networking subsystems.

The class is designed for minimal overhead. The public, mutable fields (**x**, **y**) are an intentional performance-oriented decision, eliminating the method invocation overhead of getters and setters in computationally intensive loops, such as rendering or physics updates.

Its integration with the Netty ByteBuf, including explicit Little-Endian serialization, firmly places it as a core component of the network protocol's data serialization layer. This ensures consistent representation of spatial data between the client and server, regardless of the underlying hardware architecture.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand and at high frequency. Common sources include network packet deserialization, temporary variables in mathematical calculations, or representing UI element positions.
- **Scope:** The lifecycle of a Vec2f instance is expected to be extremely short. They are typically stack-allocated or become immediately eligible for garbage collection after a single computational unit.
- **Destruction:** Managed entirely by the Java Garbage Collector. Due to their transient nature, they impose minimal pressure on the memory management system.

## Internal State & Concurrency
- **State:** The state is mutable and consists of two public float primitives: **x** and **y**. The class itself is stateless beyond these two fields.
- **Thread Safety:** This class is **not thread-safe**. Direct access to its public fields means that concurrent modification and reads will lead to race conditions.

**WARNING:** Instances of Vec2f must NOT be shared between threads without external synchronization. The intended pattern is to use them as thread-local value objects or to create new instances as needed.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vec2f(float x, float y) | constructor | O(1) | Constructs a new vector with the specified components. |
| deserialize(ByteBuf, int) | static Vec2f | O(1) | Constructs a new Vec2f by reading 8 bytes from a Netty buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the vector's 8 bytes of state into the provided Netty buffer. |

## Integration Patterns

### Standard Usage
Vec2f is primarily used for network data transfer and in-memory calculations. The serialization and deserialization methods are the core integration points with the networking layer.

```java
// Example: Deserializing player position from a network buffer
Vec2f playerPosition = Vec2f.deserialize(networkBuffer, POSITION_OFFSET);

// Example: Performing a calculation and serializing the result
Vec2f newVelocity = new Vec2f(10.5f, -3.0f);
newVelocity.serialize(outgoingPacketBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Shared State:** Do not store a single Vec2f instance in a location accessible by multiple threads. This is a direct path to data corruption and non-deterministic behavior.
- **Long-Term Storage:** Avoid holding references to Vec2f instances for extended periods. They are designed as lightweight, temporary objects, not as persistent entities.

## Data Pipeline
Vec2f acts as the data payload for any system that processes 2D spatial information. It is frequently the input and output of serialization and game logic stages.

> Flow:
> Network Byte Stream -> Netty ByteBuf Decoder -> **Vec2f** (Deserialized) -> Game Logic (Physics, AI) -> **Vec2f** (Serialized) -> Netty ByteBuf Encoder -> Network Byte Stream

