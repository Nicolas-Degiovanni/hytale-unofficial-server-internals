---
description: Architectural reference for ParticleScaleRatioConstraint
---

# ParticleScaleRatioConstraint

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum ParticleScaleRatioConstraint {
```

## Architecture & Concepts
The ParticleScaleRatioConstraint enum defines a fixed, type-safe contract for how a particle's scale should behave relative to its aspect ratio. It is a fundamental component of the network protocol layer, designed to replace ambiguous integer flags ("magic numbers") with a self-documenting and robust type.

Its primary role is to facilitate the serialization and deserialization of particle effect definitions transmitted between the client and server. The `fromValue` static method acts as a deserialization factory, converting a raw integer from a network stream into a valid, in-memory enum constant. This provides a strict validation gate at the protocol boundary, ensuring that malformed or unsupported data is rejected immediately.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) when the ParticleScaleRatioConstraint class is first loaded. This process is managed entirely by the JVM and occurs once per application lifecycle.
- **Scope:** Application-wide. As static final instances, these constants persist for the entire duration of the application's execution.
- **Destruction:** The constants are garbage collected along with their class loader when the application shuts down. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a private final integer value that is assigned at creation and can never be changed. The `VALUES` array is also static and final.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable nature and the JVM's guarantees for enum initialization, these constants can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the enum constant, intended for network serialization. |
| fromValue(int value) | ParticleScaleRatioConstraint | O(1) | Deserializes an integer from a data stream into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during protocol decoding to convert a network integer into a strongly-typed object.

```java
// How a developer should normally use this
int ratioFlag = packet.readVarInt();
ParticleScaleRatioConstraint constraint = ParticleScaleRatioConstraint.fromValue(ratioFlag);

// Now use the type-safe constant
particleEffect.setRatioConstraint(constraint);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** The ProtocolException thrown by `fromValue` is a critical signal of data corruption or a protocol version mismatch. It must not be caught and ignored, as this can lead to undefined behavior in the particle rendering system.
- **Using Ordinal for Serialization:** Never rely on the built-in `ordinal()` method for serialization. The explicit `value` field is used to guarantee that the network representation remains stable even if the declaration order of the enum constants changes in the future.

## Data Pipeline
The enum acts as a validation and translation step in the data ingress pipeline for particle effects.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> **ParticleScaleRatioConstraint.fromValue()** -> Particle Effect Builder -> Game World Renderer

