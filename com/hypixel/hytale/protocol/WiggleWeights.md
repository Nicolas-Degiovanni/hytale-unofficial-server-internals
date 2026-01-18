---
description: Architectural reference for WiggleWeights
---

# WiggleWeights

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class WiggleWeights {
```

## Architecture & Concepts
WiggleWeights is a low-level, fixed-size data structure that serves as a Data Transfer Object (DTO) within the Hytale network protocol. It is not a service or a manager; it is a pure data container with no associated logic.

Its primary role is to serialize and deserialize a specific 40-byte block of data related to procedural animation physics. The field names—such as x, y, z, roll, pitch, and their corresponding decelerations—strongly indicate its use in defining the physical properties of secondary animations, often referred to as "jiggle physics" or "wiggles". These animations apply to entity attachments like tails, cloth, or other cosmetic elements to make them react dynamically to movement.

This class is a fundamental component of the network layer, acting as a direct, memory-mapped representation of a network packet segment. Its design prioritizes performance and predictability, with a fixed memory layout and direct serialization to and from Netty ByteBufs.

## Lifecycle & Ownership
- **Creation:** WiggleWeights instances are created under two primary circumstances:
    1.  By the network protocol layer when `WiggleWeights.deserialize` is called on an incoming network buffer.
    2.  By higher-level game logic (e.g., the animation or entity systems) to populate its fields before serialization and transmission.
- **Scope:** An instance of WiggleWeights is extremely short-lived and ephemeral. It is intended to exist only for the immediate scope of a single operation—either reading its data into the game state or writing game state into a network buffer.
- **Destruction:** The object is managed by the Java garbage collector. As it holds no external resources, it is eligible for collection as soon as it falls out of scope. There is no manual destruction or cleanup process.

**WARNING:** Do not retain references to WiggleWeights instances in long-lived game state objects. They are transport objects, not state-holding components. Copy their values into the appropriate entity or animation state classes.

## Internal State & Concurrency
- **State:** The class is entirely mutable. All of its data fields are public, allowing for direct, unchecked modification. This design is intentional for high-performance serialization contexts where the object is populated and immediately consumed.
- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access. An instance of WiggleWeights must only be accessed by a single thread at any given time. Typically, this will be a Netty network thread during deserialization or the main game thread during serialization. External synchronization is required if sharing across threads is unavoidable, though this is a significant anti-pattern.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static WiggleWeights | O(1) | Constructs a new WiggleWeights object by reading 40 bytes from the provided buffer at a specific offset. |
| serialize(ByteBuf) | void | O(1) | Writes the 10 float fields of the instance into the provided buffer using little-endian byte order. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes (40) for a successful deserialization. |
| clone() | WiggleWeights | O(1) | Creates a shallow copy of the object. As all fields are primitives, this is a safe and complete copy. |
| computeSize() | int | O(1) | Returns the fixed size of the structure, which is always 40. |

## Integration Patterns

### Standard Usage
The class is designed to be used by the protocol layer. A typical deserialization flow involves validating the buffer size and then calling the static deserialize factory method.

```java
// Example of deserializing from a Netty ByteBuf
ByteBuf networkBuffer = ...;
int readOffset = ...;

// 1. Validate there is enough data to read
ValidationResult result = WiggleWeights.validateStructure(networkBuffer, readOffset);
if (!result.isOk()) {
    throw new ProtocolException(result.getErrorMessage());
}

// 2. Deserialize into a new instance
WiggleWeights weights = WiggleWeights.deserialize(networkBuffer, readOffset);

// 3. Use the data (e.g., apply to an entity's animation state)
entity.getAnimationSystem().setWigglePhysics(weights);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Retention:** Do not store WiggleWeights instances as member variables in game state classes like Entity or Player. Their purpose is for data transport only.
- **Concurrent Access:** Do not pass a WiggleWeights instance between threads. For example, do not deserialize on a network thread and then pass the same instance to the main game thread for processing without a deep copy or proper synchronization. This will introduce race conditions.
- **Partial Population:** Do not create an instance and only populate some of its fields. The serialization process unconditionally writes all 40 bytes, which would lead to uninitialized or garbage data being sent over the network.

## Data Pipeline
WiggleWeights serves as a translation point between raw network bytes and in-memory game data.

> **Incoming Flow:**
> Raw TCP Packet -> Netty ByteBuf -> **WiggleWeights.deserialize** -> In-Memory WiggleWeights Instance -> Animation System -> Entity State Update

> **Outgoing Flow:**
> Entity State Change -> Animation System -> In-Memory WiggleWeights Instance -> **WiggleWeights.serialize** -> Netty ByteBuf -> Raw TCP Packet

