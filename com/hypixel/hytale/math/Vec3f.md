---
description: Architectural reference for Vec3f
---

# Vec3f

**Package:** com.hypixel.hytale.math
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public final class Vec3f {
```

## Architecture & Concepts
The Vec3f class is a fundamental, high-performance data structure representing a 3-dimensional vector with single-precision floating-point components. It is a cornerstone of the engine's math library, used pervasively in rendering, physics, and game logic for representing positions, velocities, directions, and other 3D quantities.

Its design explicitly sacrifices encapsulation for performance. The public fields (**x**, **y**, **z**) allow for direct, low-overhead manipulation, avoiding the cost of method calls for getters and setters. This pattern is common in performance-critical game engine code where millions of vector operations may occur per frame.

The class includes direct integration with the Netty networking library via the serialize and deserialize methods, establishing its role as a primary Data Transfer Object. This allows for efficient and standardized transmission of 3D vector data over the network or for persistence to disk.

## Lifecycle & Ownership
- **Creation:** Vec3f instances are transient and created on-demand. They are instantiated by any system requiring 3D vector representation, such as physics calculations, entity position updates, or network packet deserialization. There is no central factory or manager.

- **Scope:** The lifetime of a Vec3f object is typically very short, often confined to the scope of a single method or frame update. They are treated as ephemeral value types.

- **Destruction:** Instances are managed entirely by the Java Garbage Collector. Due to their high frequency of allocation, they are a primary candidate for object pooling in performance-critical systems to reduce GC pressure, although this class itself does not implement pooling.

## Internal State & Concurrency
- **State:** The object's state is fully mutable through its public fields. It is a simple container for three float values and holds no other internal state, caches, or metadata.

- **Thread Safety:** **This class is not thread-safe.** Its mutable public fields make it inherently unsafe for concurrent access. If a Vec3f instance is shared between threads, all access must be synchronized externally. The standard and recommended practice is to avoid sharing instances altogether; each thread should operate on its own local copy.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vec3f(x, y, z) | constructor | O(1) | Creates a new vector with the specified components. |
| deserialize(buf, offset) | static Vec3f | O(1) | **Factory Method.** Deserializes 12 bytes from a Netty ByteBuf at a given offset into a new Vec3f instance. Uses little-endian byte order. |
| serialize(buf) | void | O(1) | Serializes the vector's three float components into the provided Netty ByteBuf. Writes 12 bytes using little-endian byte order. |

## Integration Patterns

### Standard Usage
Vec3f is intended for direct manipulation as a data structure. It is typically created, modified, and passed to other systems within a single operational context.

```java
// Example: Calculating a new entity position
Vec3f currentPosition = entity.getPosition(); // Returns a copy
Vec3f velocity = entity.getVelocity();
float deltaTime = frame.getDeltaTime();

// Perform calculations on a new instance or modify in-place
Vec3f nextPosition = new Vec3f(
    currentPosition.x + velocity.x * deltaTime,
    currentPosition.y + velocity.y * deltaTime,
    currentPosition.z + velocity.z * deltaTime
);

entity.setPosition(nextPosition);
```

### Anti-Patterns (Do NOT do this)
- **Sharing Instances:** Never share a single Vec3f instance across multiple threads without external locking. This will lead to race conditions and unpredictable behavior.

- **Long-Term State Storage:** Avoid storing Vec3f instances as long-lived state within components. Its mutable nature means other parts of the system could unexpectedly modify the state. If you need to store a position, it is safer to store a copy.

- **Returning Internal References:** Methods should avoid returning a direct reference to an internal Vec3f field. Always return a new instance or a copy to prevent external code from modifying the component's internal state.

## Data Pipeline
Vec3f is a critical component in the data serialization pipeline, acting as the bridge between in-memory game state and the byte-stream representation used for networking or storage.

**Outbound (Serialization):**
> Flow:
> Entity Position (Game State) -> **Vec3f** instance -> `serialize(byteBuf)` -> Network Packet

**Inbound (Deserialization):**
> Flow:
> Network Packet -> `deserialize(byteBuf, offset)` -> **Vec3f** instance -> Entity Position Update (Game State)

