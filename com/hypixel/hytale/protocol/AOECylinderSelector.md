---
description: Architectural reference for AOECylinderSelector
---

# AOECylinderSelector

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AOECylinderSelector extends Selector {
```

## Architecture & Concepts
The AOECylinderSelector is a fundamental data structure within the Hytale network protocol, designed to represent a cylindrical volume for Area-of-Effect (AOE) targeting. As a subclass of Selector, its primary purpose is to define a spatial query used by game logic to identify entities or blocks within a specific region.

This class acts as a Plain Old Java Object (POJO) whose state is tightly coupled to a fixed-size binary representation for network serialization. It is not a service or manager; it is pure data. Its design prioritizes performance and predictability in the network layer by enforcing a constant byte size, which simplifies buffer parsing and allocation. The inclusion of a nullable Vector3f for an offset allows the cylinder to be positioned relative to a source entity, providing flexibility for complex ability targeting.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1. By game logic (client or server) to define the parameters of an action, such as casting a spell or defining a trigger volume.
    2. By the protocol deserialization layer when an incoming network packet containing this structure is parsed. The static `deserialize` factory method is the entry point for this path.
- **Scope:** An AOECylinderSelector is a short-lived, transient object. Its lifetime is typically confined to the scope of a single method or network event handler. It is created, serialized or processed, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `close` methods, as it holds no native resources.

## Internal State & Concurrency
- **State:** The internal state is fully mutable. It consists of three public fields: `range`, `height`, and a nullable `offset`. The class does not contain any internal caches or derived state. Its fixed size on the wire (21 bytes) is a critical architectural constraint, achieved by writing 12 zero-bytes when the `offset` field is null.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization. It is designed to be created and used within a single thread, such as a Netty event loop thread or the main game logic thread. Concurrent modification from multiple threads will lead to data corruption and unpredictable behavior.

**WARNING:** Do not share instances of AOECylinderSelector between threads without explicit, external synchronization. The recommended pattern is to create a new instance or a clone for each distinct processing context.

## API Surface
The public API is minimal, focusing exclusively on serialization, state access, and value-based object comparisons.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | AOECylinderSelector | O(1) | **Static Factory.** Deserializes an object from a buffer at a given offset. Does not advance the buffer's reader index. |
| serialize(ByteBuf) | int | O(1) | Serializes the object's state into the provided buffer. Advances the buffer's writer index. Returns bytes written. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 21 bytes. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Utility.** Checks if the buffer has enough readable bytes for a valid structure. |
| clone() | AOECylinderSelector | O(1) | Creates a deep copy of the object, including a new instance of the offset Vector3f if present. |

## Integration Patterns

### Standard Usage
This object is almost never used in isolation. It is embedded within larger network packets to define a component of a game action. The system retrieves it from a deserialized packet and passes it to a world or physics query engine.

```java
// Example: Processing a received packet containing a selector
GameActionPacket packet = ... // Deserialized from network
AOECylinderSelector selector = packet.getTargetSelector();

if (selector != null) {
    List<Entity> targets = world.findEntitiesInCylinder(
        packet.getSourcePosition(),
        selector.range,
        selector.height,
        selector.offset
    );
    // ... apply game logic to targets
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and reuse a single AOECylinderSelector instance for multiple outgoing packets. This can lead to race conditions and state corruption if the serialization occurs on a different thread. Always create a new instance for each distinct logical operation.
- **Direct Buffer Manipulation:** Avoid calling `serialize` directly. This is the responsibility of the parent packet's serialization logic, which manages the entire buffer's state.
- **Ignoring Null Offset:** The `offset` field can be null. Logic that consumes this object must perform a null check before dereferencing it. The serialization layer handles this correctly using a bitmask, but application logic must be aware of it.

## Data Pipeline
The AOECylinderSelector is a data payload that flows through the network serialization and deserialization pipeline. It does not manage the pipeline itself.

> **Outbound Flow (Client/Server Sending Data):**
> Game Logic -> Creates `AOECylinderSelector` -> Embedded in Parent Packet -> Parent Packet `serialize` call -> **AOECylinderSelector.serialize()** -> Netty ByteBuf -> Network Socket

> **Inbound Flow (Client/Server Receiving Data):**
> Network Socket -> Netty ByteBuf -> Parent Packet `deserialize` call -> **AOECylinderSelector.deserialize()** -> Instance passed to Game Logic -> World Query

