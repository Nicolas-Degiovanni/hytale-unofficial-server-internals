---
description: Architectural reference for WorldInteraction
---

# WorldInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class WorldInteraction {
```

## Architecture & Concepts
The WorldInteraction class is a low-level Data Transfer Object (DTO) that represents a discrete event of an entity interacting with a block in the game world. It is a fundamental component of the Hytale network protocol, designed for high-performance serialization and deserialization.

This class is not a service or manager; it is a pure data container. Its primary role is to act as a structured representation of a network message that can be efficiently encoded into and decoded from a raw byte stream (Netty ByteBuf). The structure is fixed-size (20 bytes), which is critical for predictable network performance and buffer management, eliminating the need for dynamic size calculations during runtime.

The design explicitly prioritizes raw performance over encapsulation. Public fields and direct manipulation are favored to avoid method call overhead in performance-sensitive code paths like the network processing loop.

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1. By the network layer's deserialization logic when an incoming packet of this type is received. The static `deserialize` method acts as the factory.
    2. By game logic on the sending side to encapsulate an action (e.g., a player placing a block) before it is serialized and transmitted.
- **Scope:** Extremely short-lived. A WorldInteraction object typically exists only for the duration of a single transaction, such as processing one network packet or handling one game event. It is a value object, not an entity with a persistent identity.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which is usually immediately after being processed by the game logic or written to a network buffer.

## Internal State & Concurrency
- **State:** The class holds a mutable state consisting of the interacting entity's ID and optional block position and rotation. It contains no internal caches or complex state machines. The state is a direct representation of the data on the wire.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single thread, such as a Netty I/O thread or the main game logic thread. Concurrent modification from multiple threads will result in data corruption and unpredictable behavior. Any cross-thread usage requires external synchronization, which is strongly discouraged.

## API Surface
The public contract is focused entirely on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | WorldInteraction | O(1) | Static factory method. Constructs a new instance from a ByteBuf at a given offset. |
| serialize(buf) | void | O(1) | Encodes the object's state into the provided ByteBuf. |
| computeSize() | int | O(1) | Returns the fixed size (20 bytes) of the serialized object. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Checks if the buffer contains enough bytes for a valid object at the given offset. |
| clone() | WorldInteraction | O(1) | Creates a deep copy of the object. |

## Integration Patterns

### Standard Usage
The class is intended to be used as part of a network packet processing pipeline. The sender creates and populates an instance, while the receiver deserializes it to process the event.

**Receiving a message:**
```java
// In a network handler, after identifying the packet type
if (WorldInteraction.validateStructure(buffer, offset).isOk()) {
    WorldInteraction interaction = WorldInteraction.deserialize(buffer, offset);
    gameLogic.processPlayerInteraction(interaction);
}
```

**Sending a message:**
```java
// In game logic, when a player acts
WorldInteraction interaction = new WorldInteraction(
    player.getEntityId(),
    targetBlock.getPosition(),
    player.getRotation()
);

// The network layer will then call serialize
interaction.serialize(outputBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use a WorldInteraction instance for multiple distinct events. Its mutable nature makes this practice highly error-prone. Always create a new instance for each new interaction.
- **Concurrent Access:** Never share an instance between threads without explicit, external locking. This class is not designed for concurrent environments.
- **Ignoring Validation:** Bypassing `validateStructure` before calling `deserialize` can lead to `IndexOutOfBoundsException` if the network buffer is malformed or truncated. This is a critical security and stability risk.

## Data Pipeline
WorldInteraction serves as a data record that flows through the network stack.

**Outbound Flow (Client to Server):**
> Player Action -> Game Logic creates **WorldInteraction** -> `serialize()` method -> Netty ByteBuf -> Network Interface

**Inbound Flow (Server to Client):**
> Network Interface -> Netty ByteBuf -> `deserialize()` method -> **WorldInteraction** instance -> Game Logic processing

