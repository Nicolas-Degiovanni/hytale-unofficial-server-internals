---
description: Architectural reference for ParticleCollisionBlockType
---

# ParticleCollisionBlockType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility Enum

## Definition
```java
// Signature
public enum ParticleCollisionBlockType
```

## Architecture & Concepts
The ParticleCollisionBlockType enum is a type-safe data structure that defines the collision behavior of a particle with the game world. It serves as a critical component within the network protocol layer, translating between a compact integer representation used for network transmission and a readable, self-documenting constant within the game engine.

This enum's primary architectural function is to enforce data integrity during serialization and deserialization. By providing a strict contract through the fromValue method, it ensures that any malformed or out-of-range data received from a network packet is immediately rejected with a ProtocolException, preventing invalid state from propagating into the particle simulation or rendering systems.

The static VALUES array is a performance optimization, pre-caching the result of the expensive values() method call to allow for fast, allocation-free lookups in the fromValue method.

## Lifecycle & Ownership
- **Creation:** All enum constants (None, Air, Solid, All) are instantiated by the Java Virtual Machine during class loading. They are compile-time constants and exist before any game code is executed.
- **Scope:** Application-wide. These constants persist for the entire lifetime of the client or server process.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual memory management associated with this type.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant is a singleton instance with a final *value* field. Its state cannot be changed after creation.
- **Thread Safety:** This class is inherently thread-safe. As immutable singletons, its constants can be safely read and passed between any threads without synchronization. The static factory method fromValue is also thread-safe as it operates on a static final array and local variables.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for network serialization. |
| fromValue(int value) | ParticleCollisionBlockType | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used by network packet handlers and particle effect definitions to configure collision behavior.

```java
// Deserializing a particle effect from a network stream
int collisionTypeInt = buffer.readVarInt();
ParticleCollisionBlockType collisionType = ParticleCollisionBlockType.fromValue(collisionTypeInt);

// Configuring a particle system
ParticleSystem system = new ParticleSystem();
system.setCollisionBehavior(collisionType);
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Value:** Do not use the integer value for logical comparisons. This defeats the purpose of a type-safe enum and creates brittle code.
    - **BAD:** `if (type.getValue() == 2)`
    - **GOOD:** `if (type == ParticleCollisionBlockType.Solid)`
- **Reliance on Ordinal:** Never use the built-in ordinal() method. The network protocol is explicitly bound to the integer *value* field, which is stable even if the declaration order of the enum constants changes.

## Data Pipeline
This enum acts as a translation gateway between the raw network layer and the logical game simulation layer.

> **Inbound Flow (Client Receiving Data):**
> Network Packet (int) -> Protocol Buffer Decoder -> **ParticleCollisionBlockType.fromValue(int)** -> Particle System Configuration

> **Outbound Flow (Server Sending Data):**
> Particle System Configuration -> **instance.getValue()** -> Protocol Buffer Encoder -> Network Packet (int)

