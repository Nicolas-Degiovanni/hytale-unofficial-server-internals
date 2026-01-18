---
description: Architectural reference for Vec4f
---

# Vec4f

**Package:** com.hypixel.hytale.math
**Type:** Value Object / Data Structure

## Definition
```java
// Signature
public class Vec4f {
```

## Architecture & Concepts
Vec4f is a fundamental, immutable data structure representing a 4-dimensional vector using single-precision floating-point components. Its architectural role is twofold:

1.  **Core Value Type:** It serves as a foundational building block for high-level graphics, physics, and game logic systems. Common applications include representing homogeneous coordinates in 3D transformations, quaternions for rotations, or RGBA color values.
2.  **Network Data Transfer Object (DTO):** The class includes specialized serialization and deserialization methods that operate directly on Netty ByteBuf instances. This design indicates its critical role in the network protocol, providing a standardized and efficient wire format for transmitting 4D vector data between the client and server.

Its immutability is a key design choice, ensuring that instances of Vec4f are inherently thread-safe and can be passed through different systems without risk of unintended side effects.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand whenever a 4D vector representation is required. This occurs frequently during network packet processing via the static `deserialize` factory method or during game logic calculations using the public constructor.
-   **Scope:** Vec4f objects are transient and typically have a very short lifespan. Their scope is often confined to a single method or a single frame update. They are not managed by any service container or registry.
-   **Destruction:** As simple heap-allocated objects, instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced. No manual memory management is required.

## Internal State & Concurrency
-   **State:** **Immutable**. All component fields (*x, y, z, w*) are declared `public final`. Once an instance is constructed, its state cannot be altered. Any operation that would logically change the vector (e.g., addition, normalization) must result in the creation of a *new* Vec4f instance.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a Vec4f instance can be safely read by and shared across multiple threads without any external synchronization or locking mechanisms.

## API Surface
The public API is minimal, focusing exclusively on creation and network serialization. Mathematical operations are expected to be handled by external utility classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vec4f(x, y, z, w) | constructor | O(1) | Constructs a new vector from four float components. |
| deserialize(buf, offset) | static Vec4f | O(1) | Constructs a new vector by reading 16 bytes from a Netty ByteBuf at a given offset. Assumes little-endian byte order. |
| serialize(buf) | void | O(1) | Writes the vector's 16 bytes of data into a Netty ByteBuf. Uses little-endian byte order. |

## Integration Patterns

### Standard Usage
Vec4f is primarily used for data representation and network transport.

```java
// Standard creation for game logic
Vec4f color = new Vec4f(1.0f, 0.5f, 0.2f, 1.0f); // An orange color with full alpha

// Deserializing an entity's rotation quaternion from a network buffer
int dataOffset = 64; // Example offset in the buffer
Vec4f rotation = Vec4f.deserialize(networkPacketBuffer, dataOffset);

// Serializing a position vector to send over the network
ByteBuf outgoingBuffer = ...;
Vec4f position = new Vec4f(100.5f, 80.0f, 25.25f, 1.0f);
position.serialize(outgoingBuffer);
```

### Anti-Patterns (Do NOT do this)
-   **High-Frequency Allocation:** Avoid creating new Vec4f instances inside tight, performance-critical loops that run thousands of times per frame. The resulting GC pressure can lead to performance stutter. For such scenarios, use mutable vector types or object pooling systems if available elsewhere in the engine.
-   **Attempted Modification:** Do not retain a reference to a Vec4f with the expectation of modifying it. Its immutable design requires you to create a new instance for any new value.

## Data Pipeline
Vec4f acts as a data container at the boundary between the game state and the network layer.

**Serialization Flow (Outgoing Data):**
> Game Logic State -> **Vec4f** instance -> `serialize()` -> Netty ByteBuf -> Network Encoder -> TCP/UDP Socket

**Deserialization Flow (Incoming Data):**
> TCP/UDP Socket -> Network Decoder -> Netty ByteBuf -> `deserialize()` -> **Vec4f** instance -> Game Logic State

