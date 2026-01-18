---
description: Architectural reference for EntityMatcher
---

# EntityMatcher

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class EntityMatcher {
```

## Architecture & Concepts
The EntityMatcher is a low-level data structure that functions as a Data Transfer Object (DTO) within the Hytale network protocol. Its primary role is to define a simple, serializable rule for filtering or targeting entities within the game world. It is not a standalone service but rather a component field embedded within more complex network packets.

This class acts as a fundamental building block for any network operation that needs to specify a target set of entities, such as applying an effect, sending a command, or querying for game state. The design prioritizes serialization performance and a minimal memory footprint, evident from its fixed size and direct byte manipulation methods. It encapsulates a simple condition: a *type* of entity to match (e.g., Server, Player, All) and an *inversion* flag to negate the match (e.g., "all entities *except* players").

## Lifecycle & Ownership
- **Creation:** An EntityMatcher instance is created under two primary circumstances:
    1. **Deserialization:** The static factory method *deserialize* instantiates the object when reading an incoming network packet from a Netty ByteBuf. This is the most common creation path on the receiving end of a connection.
    2. **Direct Instantiation:** Game logic instantiates it directly via its constructor when building an outgoing network packet that requires an entity-targeting rule.
- **Scope:** The object's lifetime is extremely short and bound to the lifecycle of its containing network packet. It is created, used for a single operation, and then becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction logic. The Java Garbage Collector reclaims the memory once the parent packet object is no longer referenced, typically after a single game tick or network event processing cycle.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of two fields: *type* (an EntityMatcherType enum) and *invert* (a boolean). The class is a simple data container with no caching or complex state management. Its state directly reflects the data serialized to or from the network buffer.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization. It is designed to be created and accessed by a single thread, typically a Netty I/O worker thread or the main game loop thread.

**WARNING:** Sharing an EntityMatcher instance across multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or modify them from multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | EntityMatcher | O(1) | **Static Factory.** Constructs an instance from a network buffer at a given offset. |
| serialize(buf) | void | O(1) | Writes the object's state into the provided network buffer. |
| computeSize() | int | O(1) | Returns the fixed byte size (2) of the serialized object. |
| validateStructure(buffer, offset) | ValidationResult | O(1) | **Static.** Checks if a buffer has enough readable bytes to contain a valid instance. |
| clone() | EntityMatcher | O(1) | Creates a shallow copy of the instance. |

## Integration Patterns

### Standard Usage
EntityMatcher is not used in isolation. It is set as a property of a larger packet object before the packet is sent over the network. On the receiving end, it is read from the packet to determine the logic for entity selection.

```java
// Example: Building a packet that targets all non-player entities
SomeGamePacket packet = new SomeGamePacket();
EntityMatcher matcher = new EntityMatcher(EntityMatcherType.Player, true); // Type: Player, Invert: true
packet.setTarget(matcher);

networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse an EntityMatcher instance across multiple, logically distinct packets. This can lead to state corruption if one packet's logic inadvertently modifies the matcher before another packet is serialized. Always create a new instance for each new packet.
- **Cross-Thread Modification:** Do not create an EntityMatcher on one thread and pass it to another for modification without proper synchronization. The object is not designed for concurrent access.
- **Treating as a Service:** This is a data object, not a service or manager. Do not attempt to register it in a dependency injection context or hold long-lived references to it.

## Data Pipeline
The EntityMatcher is a payload component within the network data stream. Its flow is simple and direct, representing a single step in the packet serialization and deserialization process.

> **Ingress Flow:**
> Raw TCP Stream -> Netty ByteBuf -> **EntityMatcher.deserialize** -> Parent Packet Object -> Game Event Logic

> **Egress Flow:**
> Game Logic -> new **EntityMatcher()** -> Parent Packet Object -> **EntityMatcher.serialize** -> Netty ByteBuf -> Raw TCP Stream

