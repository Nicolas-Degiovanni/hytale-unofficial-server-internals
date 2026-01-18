---
description: Architectural reference for CanMoveType
---

# CanMoveType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum CanMoveType {
```

## Architecture & Concepts
CanMoveType is a fundamental enumeration within the Hytale network protocol layer. It serves as a strict data contract, defining the possible movement states for a game entity. Its primary purpose is to provide a type-safe and highly optimized representation of this state for serialization and deserialization within network packets.

Instead of transmitting verbose strings, the protocol uses the integer representation of this enum, minimizing packet size and reducing processing overhead. The two defined states are:

*   **AttachedToLocalPlayer:** Indicates an entity whose movement is directly coupled with or controlled by the local player. This is typical for mounts, pets, or other companion entities.
*   **Always:** Indicates an entity with independent movement logic, controlled by server-side AI or other game systems. This applies to most NPCs, monsters, and environmental creatures.

This component is not a service or manager; it is a pure value object used to model a specific, constrained piece of game state.

## Lifecycle & Ownership
- **Creation:** All instances of CanMoveType are constants created by the Java Virtual Machine during class loading. They are not instantiated at runtime by application code.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the client or server process.
- **Destruction:** Instances are reclaimed by the JVM only upon application shutdown. There is no manual destruction or garbage collection concern.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant is a singleton instance with a final, private integer field. Its state cannot be modified after creation. The static VALUES array is a read-only cache populated at class-load time.
- **Thread Safety:** Inherently thread-safe. As an immutable, globally accessible constant, CanMoveType can be safely read from any thread without locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for network serialization. |
| fromValue(int value) | CanMoveType | O(1) | Deserializes an integer from a network packet into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary integration point is during the serialization and deserialization of network packets that contain entity state. Use getValue to write to a packet buffer and fromValue to read from it.

```java
// Example: Deserializing an entity update packet
int rawMoveType = buffer.readInt();
CanMoveType moveType = CanMoveType.fromValue(rawMoveType);
entity.setMovementState(moveType);

// Example: Serializing an entity state packet
CanMoveType currentMoveType = entity.getCanMoveType();
buffer.writeInt(currentMoveType.getValue());
```

### Anti-Patterns (Do NOT do this)
- **String-Based Serialization:** Do not use the enum's string name for serialization (e.g., via `name()`). The network protocol is strictly defined to use the integer value for efficiency. Relying on the string name is brittle and inefficient.
- **Unsafe Deserialization:** Do not call `fromValue` without a try-catch block if the data source is untrusted. A malicious or corrupted packet could provide an invalid integer, which would throw a ProtocolException and could crash the packet processing thread if unhandled.
- **Reflection:** Do not attempt to create new instances of this enum using reflection. This violates the fundamental contract of enumerations and will lead to unpredictable system behavior.

## Data Pipeline
CanMoveType is not a processing stage in a pipeline; rather, it is the data that flows through it. It represents a piece of state that is serialized, transmitted, and deserialized.

> Flow:
> Server Entity State -> **CanMoveType.getValue()** -> Packet Serializer -> Network Layer -> Packet Deserializer -> **CanMoveType.fromValue()** -> Client Entity State

