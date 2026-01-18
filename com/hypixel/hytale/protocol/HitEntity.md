---
description: Architectural reference for HitEntity, a core data structure in the Hytale network protocol for representing entity interactions.
---

# HitEntity

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class HitEntity {
```

## Architecture & Concepts
The HitEntity class is a Plain Old Java Object (POJO) that serves as a Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or a manager; its sole purpose is to provide a structured, in-memory representation of a binary network message related to entity collision or interaction events.

This class acts as the concrete schema for a specific data packet. It contains the logic to serialize itself into a Netty ByteBuf for network transmission and to deserialize a ByteBuf back into a HitEntity object upon receipt.

The design is heavily optimized for low-latency, high-throughput networking. This is evident from its direct manipulation of byte buffers, use of fixed-size data blocks, a bitmask for handling nullable fields, and variable-length integers (VarInt) for encoding array lengths efficiently. This approach minimizes both memory allocation and network bandwidth.

## Lifecycle & Ownership
- **Creation:** A HitEntity instance is created in one of two scenarios:
    1.  **Inbound:** The static factory method `deserialize` is called by a network pipeline handler when an incoming ByteBuf corresponding to this message type is received.
    2.  **Outbound:** Game logic (e.g., a combat system) instantiates it directly using its constructor to build a message that needs to be sent to another client or the server.

- **Scope:** The object's lifetime is exceptionally short and bound to the scope of a single network operation. It is created, processed, and then immediately becomes eligible for garbage collection. It holds no references to long-lived services and is not registered in any context.

- **Destruction:** Cleanup is managed entirely by the Java Garbage Collector. Once the network handler or game logic method completes, the stack-local reference to the HitEntity instance is lost, and the object is reclaimed.

## Internal State & Concurrency
- **State:** The state of a HitEntity object is fully defined by its public fields `next` and `matchers`. The class is fundamentally **mutable** and is designed to be populated once upon creation. It performs no internal caching or state management beyond the data it directly represents.

- **Thread Safety:** This class is **not thread-safe**. All of its fields are public and non-final. It is designed for use within a single-threaded execution model, such as a Netty I/O thread or a main game loop thread. Concurrent access from multiple threads without external locking mechanisms will result in data corruption and undefined behavior.

**WARNING:** Never share a HitEntity instance across threads. If data must be passed to another thread, create a deep copy using the `clone` method.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static HitEntity | O(N) | Constructs a new HitEntity from a ByteBuf. Throws ProtocolException on malformed data. N is the number of matchers. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the byte size of a serialized HitEntity within a buffer without full deserialization. Critical for skipping messages. |
| computeSize() | int | O(1) | Calculates the required byte size to serialize the current instance. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a fast, low-cost check to validate if a buffer likely contains a valid HitEntity. Does not perform a full parse. |
| clone() | HitEntity | O(N) | Creates a deep copy of the object and its internal EntityMatcher array. |

## Integration Patterns

### Standard Usage
A HitEntity is typically created by game logic, serialized into a buffer, and written to a network channel.

```java
// Example: A game system creates and prepares a HitEntity message for sending.
EntityMatcher[] targetMatchers = buildTargetMatchers();
int nextTargetId = findNextTarget();

// 1. Instantiate the DTO with game state data.
HitEntity hitMessage = new HitEntity(nextTargetId, targetMatchers);

// 2. Allocate a buffer and serialize the message.
ByteBuf networkBuffer = getChannel().alloc().buffer(hitMessage.computeSize());
hitMessage.serialize(networkBuffer);

// 3. Write the buffer to the network pipeline.
getChannel().writeAndFlush(networkBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Object Reuse:** Do not modify a HitEntity instance after it has been serialized or hold onto it for later use. Treat it as an immutable record after creation. Reusing instances can lead to unpredictable behavior, especially if the internal arrays are modified elsewhere.

- **Unsafe Deserialization:** Never call `deserialize` on a buffer from an untrusted source without first calling `validateStructure`. Failing to do so exposes the server to potential buffer overflow attacks or `ProtocolException` crashes from malformed packets.

- **Concurrent Modification:** Do not pass a HitEntity instance to another thread while the original thread still holds a reference. If data must be shared, use the `clone` method to provide a safe, deep copy.

## Data Pipeline
HitEntity acts as a translation layer between raw bytes on the network and structured objects in the game engine.

**Inbound Flow (Receiving Data):**
> Raw ByteBuf from Netty -> `HitEntity.deserialize` -> **HitEntity Instance** -> Game Logic (e.g., Combat System)

**Outbound Flow (Sending Data):**
> Game Logic -> `new HitEntity()` -> **HitEntity Instance** -> `hitEntity.serialize` -> Raw ByteBuf for Netty

