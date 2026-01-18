---
description: Architectural reference for RaycastSelector
---

# RaycastSelector

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class RaycastSelector extends Selector {
```

## Architecture & Concepts
The RaycastSelector is a highly-optimized, fixed-size data structure that defines the parameters for a world query. It is not a service or manager, but rather a fundamental component of the Hytale network protocol, acting as a data contract for specifying raycasting operations between the client and server.

Its primary architectural role is to serve as a Plain Old Java Object (POJO) for serialization and deserialization. The design prioritizes performance and low memory overhead, which is evident from its rigid 23-byte binary layout and direct manipulation of Netty ByteBufs. This avoids the overhead of more flexible serialization formats like JSON or Protocol Buffers, which is critical for high-frequency game state communication.

As a subclass of Selector, it participates in a larger polymorphic system for defining different types of world queries, allowing game logic to handle various selection mechanisms through a common interface.

## Lifecycle & Ownership
- **Creation:** A RaycastSelector is typically instantiated in one of two scenarios:
    1.  By the network layer when a corresponding packet is received, calling the static *deserialize* method to construct the object from a raw ByteBuf.
    2.  By game logic systems (e.g., player interaction controllers) that need to build a query to be sent over the network.
- **Scope:** The object's lifetime is ephemeral. It is designed to exist only for the duration of a single transaction, such as the processing of one network packet or the execution of a single game tick's logic. It is a value object, not a persistent entity.
- **Destruction:** The object is managed by the Java garbage collector and is reclaimed once it is no longer referenced. There are no manual cleanup or disposal methods. Ownership is transferred by value, either through direct instantiation or by using the *clone* method.

## Internal State & Concurrency
- **State:** The RaycastSelector is a mutable data container. All of its public fields can be modified directly after instantiation. It holds the complete state for a single raycast query and does not cache any external data.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking mechanisms and is intended for use within a single-threaded context, such as a Netty event loop thread or the main game engine thread.

    **WARNING:** Concurrent modification of a RaycastSelector instance from multiple threads will result in data corruption and undefined behavior. It must not be shared across threads without external synchronization, which is considered an anti-pattern for this class.

## API Surface
The public API is focused exclusively on serialization, data access, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static RaycastSelector | O(1) | Constructs a new RaycastSelector by reading 23 bytes from the given buffer at a specific offset. |
| serialize(ByteBuf) | int | O(1) | Writes the object's state into a 23-byte block in the provided buffer. Returns the number of bytes written. |
| computeSize() | int | O(1) | Returns the constant binary size of the structure, which is always 23. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid structure. |
| clone() | RaycastSelector | O(1) | Creates a deep copy of the object, including a new instance of the Vector3f offset if present. |

## Integration Patterns

### Standard Usage
The canonical use case involves deserializing the object from a network buffer, passing it to a game system for processing, and then discarding it.

```java
// Executed within a network handler or game loop
// The ByteBuf contains the incoming packet data
if (RaycastSelector.validateStructure(buffer, offset).isOk()) {
    RaycastSelector query = RaycastSelector.deserialize(buffer, offset);

    // Pass the immutable query parameters to the physics engine
    RaycastResult result = world.getPhysicsEngine().performRaycast(query);

    // Process the result...
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and reuse a RaycastSelector instance across multiple distinct operations. Its mutable nature makes it easy to forget to reset a field, leading to subtle and hard-to-diagnose bugs. Always create a new instance for a new query.
- **Shared State:** Do not store a RaycastSelector as a field in a long-lived singleton or service. This creates shared mutable state, which is a primary source of concurrency issues.

## Data Pipeline
The RaycastSelector serves as a data record that flows from the network layer into the core game simulation.

> Flow:
> Inbound Network ByteBuf -> Protocol Decoder -> **RaycastSelector.deserialize** -> Game Logic (WorldPhysicsEngine) -> RaycastResult -> Outbound Network Packet

