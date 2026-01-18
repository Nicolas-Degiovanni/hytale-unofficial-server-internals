---
description: Architectural reference for Vector2i
---

# Vector2i

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class Vector2i {
```

## Architecture & Concepts
The Vector2i class is a fundamental Data Transfer Object (DTO) used within the Hytale network protocol. It represents a two-dimensional integer vector, primarily for world coordinates, grid positions, or UI element locations.

Architecturally, this class is designed for extreme performance and memory efficiency. It is not a general-purpose mathematical vector class; its primary role is to define a fixed-size, 8-byte binary representation for a coordinate pair that can be rapidly serialized and deserialized. The presence of static methods like *deserialize* and *validateStructure*, along with direct manipulation of Netty's ByteBuf, indicates its tight coupling with the low-level networking layer.

This design choice bypasses heavier, reflection-based serialization frameworks in favor of a rigid, contract-based binary format, which is critical for a high-throughput game server. The public fields, while violating traditional encapsulation, are a deliberate performance optimization to eliminate method call overhead in performance-critical code paths like the game loop or network processing threads.

## Lifecycle & Ownership
- **Creation:** Vector2i instances are created on-demand. This occurs either within game logic (e.g., `new Vector2i(10, 20)`) when preparing data to be sent, or by the protocol layer's decoders when `Vector2i.deserialize` is called to reconstruct an object from an incoming network buffer.
- **Scope:** The lifetime of a Vector2i object is typically very short. It is a value type, intended to exist only for the duration of a single transaction or event. For example, it may be a field within a larger packet object, and its lifetime is bound to that of the containing packet.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced. There are no manual destruction or resource release patterns associated with this class.

## Internal State & Concurrency
- **State:** The class is fully mutable. Its core state consists of the public integer fields *x* and *y*. This design prioritizes performance and ease of use over immutability.
- **Thread Safety:** **This class is not thread-safe.** The public, mutable fields make it susceptible to race conditions if an instance is shared and modified concurrently by multiple threads. It is designed to be used within a single-threaded context, such as a Netty event loop or the main game thread. Any cross-thread usage must be managed with external synchronization, which is strongly discouraged.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vector2i(int x, int y) | constructor | O(1) | Creates a new instance with specified coordinates. |
| serialize(ByteBuf buf) | void | O(1) | Writes the 8-byte binary representation into the buffer. |
| deserialize(ByteBuf buf, int offset) | static Vector2i | O(1) | Reads 8 bytes from the buffer at an offset and creates a new instance. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a bounds check to ensure 8 bytes are readable from the offset. |
| clone() | Vector2i | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
Vector2i is almost exclusively used as part of the network protocol. Developers should interact with it as a data field within larger packet objects, not as a standalone entity.

**Serialization (Outbound Packet Construction)**
```java
// Example: Creating a packet to send to the server
PlayerMovePacket packet = new PlayerMovePacket();
packet.newPosition = new Vector2i(100, 250);

// The network layer will later call serialize on the packet,
// which in turn calls serialize on the Vector2i field.
ByteBuf buffer = ...;
packet.serialize(buffer);
```

**Deserialization (Inbound Packet Handling)**
```java
// The network layer decodes a buffer into a packet object
PlayerMovePacket packet = PlayerMovePacket.deserialize(buffer, 0);

// Game logic can then access the deserialized data
int playerX = packet.newPosition.x;
int playerY = packet.newPosition.y;
```

### Anti-Patterns (Do NOT do this)
- **Sharing Instances:** Do not share a single Vector2i instance across multiple threads or long-lived objects. Its mutability can lead to unpredictable state corruption. Always create a new instance or use clone() if you need to pass its value elsewhere.
- **Modification After Serialization:** Modifying a Vector2i object after its parent packet has been serialized will have no effect on the data sent over the network. This can cause severe state desynchronization bugs.
- **Performance-Blind Instantiation:** Avoid creating new Vector2i instances inside tight, performance-critical loops. While cheap, allocations can add up. In such scenarios, prefer to reuse a single instance by updating its fields.

## Data Pipeline

The Vector2i class is a payload component within the network data pipeline. It does not process data itself but rather represents a piece of data being processed.

> **Outbound Flow:**
> Game Logic -> `new Vector2i()` -> Packet Field -> **Vector2i.serialize()** -> Netty ByteBuf -> Network Socket

> **Inbound Flow:**
> Network Socket -> Netty ByteBuf -> Packet Decoder -> **Vector2i.deserialize()** -> Packet Field -> Game Logic

