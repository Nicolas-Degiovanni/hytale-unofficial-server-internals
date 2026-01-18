---
description: Architectural reference for Hitbox
---

# Hitbox

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class Hitbox {
```

## Architecture & Concepts
The Hitbox class is a fundamental data structure representing an Axis-Aligned Bounding Box (AABB). It is not a service or manager, but rather a plain data object designed for high-performance scenarios, particularly network serialization and physics calculations.

Its primary role is to define the physical volume of an entity within the game world. The design, characterized by public fields and static serialization methods, is heavily optimized for the engine's network protocol layer. By avoiding method call overhead for field access (getters/setters) and providing direct binary translation to and from a Netty ByteBuf, it minimizes both CPU cost and garbage collection pressure during the critical path of packet processing.

The class structure, with its fixed size constants and serialization logic, strongly indicates it is part of an automated or semi-automated protocol generation system. It serves as a data contract between the client and server for communicating spatial information.

## Lifecycle & Ownership
- **Creation:** Hitbox instances are highly transient and created on-demand. The primary sources of instantiation are:
    1.  **Network Deserialization:** The static `deserialize` method is called by the network protocol decoder when a packet containing hitbox data is received.
    2.  **Game Logic:** Game systems create new instances to define entity properties, often using the parameterized constructor: `new Hitbox(minX, ..., maxZ)`.
    3.  **State Replication:** The copy constructor or `clone` method is used to create safe copies for physics prediction or to avoid state corruption across threads.

- **Scope:** The lifetime of a Hitbox is typically short and confined to a specific operation, such as a single game tick update, the processing of one network packet, or a temporary collision check. They are not intended to be long-lived, shared objects.

- **Destruction:** Instances are managed entirely by the Java Garbage Collector. Given their small, fixed size and transient nature, they are efficiently collected with minimal performance impact.

## Internal State & Concurrency
- **State:** The state of a Hitbox is entirely defined by its six public float fields. It is a fully **mutable** object. This design choice prioritizes performance in single-threaded contexts, such as the main game loop, where direct field manipulation is faster than method calls.

- **Thread Safety:** **This class is not thread-safe.** Direct access to its public fields makes it vulnerable to race conditions and memory consistency errors if an instance is shared and modified across multiple threads.

    **WARNING:** Any system that shares Hitbox instances between threads must implement its own external synchronization. The recommended and safest pattern is to pass immutable copies of the Hitbox (created via `clone()` or the copy constructor) to other threads.

## API Surface
The public API is focused on serialization, validation, and state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Hitbox | O(1) | Constructs a new Hitbox by reading 24 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the six float fields into the provided buffer using little-endian byte order. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer contains enough readable bytes to deserialize a Hitbox. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object in bytes, which is always 24. |
| clone() | Hitbox | O(1) | Creates and returns a deep copy of the instance. |

## Integration Patterns

### Standard Usage
A Hitbox is typically used to define an entity's bounds and then serialized as part of a larger network packet.

```java
// Example: Defining a new entity's physical volume
Hitbox entityBounds = new Hitbox(-0.4f, 0.0f, -0.4f, 0.4f, 1.8f, 0.4f);

// Example: Serializing the hitbox into a network buffer
ByteBuf networkBuffer = getPacketBuffer();
entityBounds.serialize(networkBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not store a reference to a single Hitbox instance in multiple systems that can modify it. This will lead to unpredictable behavior and race conditions.

    ```java
    // BAD: Physics and AI systems modify the same object reference
    Hitbox sharedBox = myEntity.getHitbox();
    physicsThread.process(sharedBox); // Modifies the box
    aiThread.process(sharedBox);      // Also modifies, causing a race condition
    
    // GOOD: Provide copies to each system
    Hitbox physicsCopy = myEntity.getHitbox().clone();
    physicsThread.process(physicsCopy);
    Hitbox aiCopy = myEntity.getHitbox().clone();
    aiThread.process(aiCopy);
    ```

- **Modification During Serialization:** Modifying a Hitbox's fields on one thread while another thread is calling `serialize` on the same instance will result in a corrupted data stream.

## Data Pipeline
As a data object, Hitbox is a payload that flows through different engine systems. It does not actively process data itself.

**Outbound (Serialization):**
> Flow:
> Game Logic (Entity State) -> **Hitbox** instance created/updated -> `serialize()` method called -> Netty ByteBuf -> Network Socket

**Inbound (Deserialization):**
> Flow:
> Network Socket -> Netty ByteBuf -> `deserialize()` method called -> New **Hitbox** instance -> Game Logic (Entity Creation, Physics Update)

