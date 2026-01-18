---
description: Architectural reference for PhysicsType
---

# PhysicsType

**Package:** com.hypixel.hytale.protocol
**Type:** Value Type / Utility

## Definition
```java
// Signature
public enum PhysicsType {
```

## Architecture & Concepts
The PhysicsType enum is a type-safe constant that represents a distinct physics simulation model within the Hytale protocol. Its primary function is to provide a stable, integer-backed identifier for different physics behaviors, which is essential for network serialization and deserialization.

This class acts as a critical translation layer between the low-level network protocol, which deals in integers, and the high-level game engine, which requires explicit, readable types. By mapping a name like *Standard* to a fixed integer value, it eliminates the use of "magic numbers" in the codebase, improving readability and maintainability.

The static `fromValue` method serves as a deserialization factory and a validation gate. It ensures that any integer received from a network stream corresponds to a valid, known physics model, throwing a ProtocolException for invalid data. This prevents corrupted or malicious packets from propagating invalid state into the game simulation.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during the initial class loading phase. This process is guaranteed to happen only once.
- **Scope:** Application-wide. The instances, such as PhysicsType.Standard, are static constants that persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state of each enum constant is final and established at compile time. The static `VALUES` array, a cache for the `values()` method, is populated once during class initialization and is not modified thereafter.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature and the JVM's guarantees for enum initialization, this class can be safely accessed and used by multiple threads simultaneously without any external synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value associated with the enum constant, intended for network serialization. |
| fromValue(int value) | PhysicsType | O(1) | A static factory method that converts an integer from a network stream into the corresponding PhysicsType. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a value from a network buffer and serializing it back. The `fromValue` and `getValue` methods form a symmetric pair for this purpose.

```java
// DESERIALIZATION: Reading from a network source
int rawType = networkBuffer.readVarInt();
PhysicsType physicsModel = PhysicsType.fromValue(rawType);
entity.setPhysics(physicsModel);

// SERIALIZATION: Writing to a network destination
PhysicsType currentModel = entity.getPhysics();
networkBuffer.writeVarInt(currentModel.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Never use the built-in `ordinal()` method for serialization. The integer value returned by `ordinal()` is dependent on the declaration order of the constants. If a new constant is added in the future, the ordinals of all subsequent constants will shift, breaking network compatibility. Always use the explicit `getValue()` method.
- **Invalid Value Handling:** Do not wrap calls to `fromValue` in a generic `try-catch (Exception e)`. The `ProtocolException` it throws is a specific, unrecoverable error indicating a corrupt or incompatible network stream. It should be caught at the highest level of the packet processing loop to terminate the connection or discard the packet.

## Data Pipeline
The PhysicsType enum is a key component in the data marshalling pipeline for entity physics state.

> **Inbound Flow (Deserialization):**
> Network Packet -> Protocol Decoder -> Raw Integer ID -> **PhysicsType.fromValue()** -> Game Entity State

> **Outbound Flow (Serialization):**
> Game Entity State -> PhysicsType Instance -> **getValue()** -> Raw Integer ID -> Protocol Encoder -> Network Packet

