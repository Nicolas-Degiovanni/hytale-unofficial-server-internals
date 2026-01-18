---
description: Architectural reference for ParticleAttractor
---

# ParticleAttractor

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ParticleAttractor {
```

## Architecture & Concepts
The ParticleAttractor is a Plain Old Java Object (POJO) that functions as a **Protocol Data Structure**. Its primary role is to represent the physical properties of an attractor point within the particle simulation system for network replication. It is not a service or manager, but a pure data container designed for high-throughput serialization and deserialization.

The key architectural decision for this class is its **fixed-size binary layout**. Every ParticleAttractor instance serializes to a consistent 85-byte block. This design choice prioritizes performance over data size efficiency. By enforcing a fixed size, the network layer can perform highly optimized, zero-overhead reads and writes without needing to parse variable-length fields or size prefixes.

To accommodate nullable fields like Vector3f within this fixed structure, the class employs a bitmask pattern. The first byte of the serialized block is a `nullBits` field, where each bit flags the presence or absence of a corresponding nullable object. If an object is null, its allocated space in the byte block is padded with zeros, preserving the fixed-offset layout for subsequent fields.

## Lifecycle & Ownership
- **Creation:** A ParticleAttractor instance is created under two primary circumstances:
    1.  **Inbound:** By the network protocol layer, which calls the static `deserialize` factory method to construct an object from an incoming Netty ByteBuf.
    2.  **Outbound:** By game logic (e.g., the particle or physics engine) which instantiates a new attractor via its constructor to define its properties before serialization.

- **Scope:** This object is fundamentally **transient and short-lived**. Its lifetime is typically confined to the scope of a single network packet's processing or a single game-tick update. It is not designed to be cached or persist between major state changes.

- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as it falls out of the scope of the method that created or received it. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state is entirely **mutable**. All public fields are directly accessible and can be modified after instantiation. The class is a simple data aggregate and performs no internal caching or complex state management.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization primitives. It is designed to be owned and operated on by a single thread at a time, such as a Netty I/O thread during deserialization or the main game thread during simulation.

    **WARNING:** Sharing a ParticleAttractor instance across multiple threads without external synchronization will result in data corruption and undefined behavior.

## API Surface
The public API is focused exclusively on data transport and manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ParticleAttractor | O(1) | **Static Factory.** Constructs an instance from a fixed-size block in a network buffer. |
| serialize(ByteBuf) | void | O(1) | Serializes the object's state into the provided network buffer according to its fixed layout. |
| computeSize() | int | O(1) | Returns the constant size (85 bytes) of the serialized object. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Performs a pre-flight check to ensure a buffer has enough readable bytes for deserialization. |
| clone() | ParticleAttractor | O(1) | Creates a deep copy of the object, including new instances of its Vector3f fields. |

## Integration Patterns

### Standard Usage
The class is intended to be used as part of a network serialization pipeline. Game logic populates the structure, and the network layer serializes it, or vice-versa.

```java
// Example: Deserializing from a network buffer
ByteBuf networkBuffer = ...;
ParticleAttractor attractor = ParticleAttractor.deserialize(networkBuffer, 0);

// Use the attractor in game logic
particleSystem.applyAttractor(attractor);
```

```java
// Example: Serializing for network transmission
ParticleAttractor newAttractor = new ParticleAttractor();
newAttractor.position = new Vector3f(10, 50, 10);
newAttractor.radius = 25.0f;
newAttractor.radialImpulse = 100.0f;

// Serialize into a buffer to be sent
ByteBuf outBuffer = ...;
newAttractor.serialize(outBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a ParticleAttractor instance across multiple network messages or physics ticks without explicitly resetting all of its fields. The cost of instantiation is trivial, and creating new instances prevents bugs from stale data.
- **Concurrent Modification:** Do not pass an instance to another thread for processing while the original thread might still be modifying it. Either pass a clone or ensure a proper happens-before relationship with memory visibility guarantees.
- **Manual Serialization:** Do not attempt to read or write the fields to a buffer manually. The `serialize` and `deserialize` methods correctly handle the `nullBits` bitmask, byte order (Little Endian), and zero-padding. Bypassing them will break the protocol.

## Data Pipeline
The ParticleAttractor serves as a data record that flows between the game engine and the network stack.

**Outbound Flow (Game to Network):**
> Game Logic -> Creates and populates **ParticleAttractor** -> `serialize()` -> Netty ByteBuf -> Network Stack

**Inbound Flow (Network to Game):**
> Network Stack -> Netty ByteBuf -> `deserialize()` -> Creates **ParticleAttractor** -> Game Logic

