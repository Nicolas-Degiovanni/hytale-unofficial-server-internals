---
description: Architectural reference for AppliedForce
---

# AppliedForce

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class AppliedForce {
```

## Architecture & Concepts
The AppliedForce class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It is not a service or manager, but rather a fundamental data container that represents a physics-based force event within the Hytale game protocol.

Its primary architectural role is to serve as a concrete, serializable representation of a force vector applied to an entity. The design is heavily optimized for predictable network performance, mandating a fixed 18-byte memory and wire footprint. This fixed-size structure eliminates the need for dynamic size calculations during serialization, reducing CPU overhead and ensuring consistent packet sizing, which is critical for real-time game networking.

This class exists at the boundary between the core game logic (e.g., the physics engine) and the low-level network transport layer (e.g., Netty).

## Lifecycle & Ownership
- **Creation:** An AppliedForce instance is created under two circumstances:
    1.  By the network protocol layer, specifically via the static `deserialize` method when an incoming network packet is being parsed.
    2.  By high-level game systems (e.g., a physics or combat system) to encapsulate a force that needs to be transmitted to other clients or the server.

- **Scope:** The lifetime of an AppliedForce object is exceptionally short and ephemeral. It is intended to exist only for the duration of a single transaction, such as one serialization event or one game tick processing cycle.

- **Destruction:** The object is managed by the Java Garbage Collector. Due to its short scope, it becomes eligible for garbage collection almost immediately after being serialized to a network buffer or after its data has been consumed by a game system. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state is fully mutable, with public fields for direct access. This design choice prioritizes raw performance over encapsulation, a common trade-off in performance-critical network DTOs. The object is a simple data aggregate and does not maintain any internal cache or complex state.

- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. Concurrent access from multiple threads without external synchronization will result in race conditions, data corruption, and undefined behavior.

    **WARNING:** Never share an AppliedForce instance across threads. If a force event needs to be passed between threads, create a deep copy using the provided copy constructor or the `clone` method.

## API Surface
The public API is minimal and focused entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | AppliedForce | O(1) | **Static Factory.** Constructs an AppliedForce instance by reading 18 bytes from the provided buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a fixed 18-byte block in the target buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 18. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Performs a pre-check to ensure a buffer contains enough readable bytes for a valid structure. |

## Integration Patterns

### Standard Usage
AppliedForce is not retrieved from a service registry. It is either instantiated directly by game logic for outbound messages or created by the network layer during deserialization for inbound messages.

```java
// Example: Creating and serializing an outbound force event
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

// 1. Game logic creates the force data
Vector3f direction = new Vector3f(0.0f, 1.0f, 0.0f);
AppliedForce jumpForce = new AppliedForce(direction, false, 50.5f);

// 2. The network layer serializes it into a buffer
ByteBuf networkBuffer = Unpooled.buffer(jumpForce.computeSize());
jumpForce.serialize(networkBuffer);

// networkBuffer is now ready to be sent over the wire
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not retain references to AppliedForce instances across multiple game ticks or network flushes. They represent a point-in-time event and should be discarded after use.
- **Concurrent Modification:** Do not write to an AppliedForce object from one thread while another thread is reading from it or serializing it. This will lead to corrupted network data.
- **Reusing Instances for Different Events:** Do not modify and reuse the same AppliedForce instance for multiple distinct force events. Always create a new instance to ensure data integrity.

## Data Pipeline
AppliedForce acts as a data payload within the broader network pipeline. It does not process data itself; rather, it is the data being processed.

**Outbound (Client to Server):**
> Flow:
> Physics System -> **AppliedForce (new instance)** -> Protocol Serializer -> Netty ByteBuf -> Network Socket

**Inbound (Server to Client):**
> Flow:
> Network Socket -> Netty ByteBuf -> Protocol Deserializer -> **AppliedForce (new instance)** -> Entity Component System

