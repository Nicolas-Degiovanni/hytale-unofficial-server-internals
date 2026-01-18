---
description: Architectural reference for NoiseType
---

# NoiseType

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum NoiseType {
```

## Architecture & Concepts
The NoiseType enum serves as a strict, type-safe contract for defining noise generation algorithms used within the Hytale engine, particularly for procedural world generation. Its primary architectural role is to provide a robust and efficient mapping between a human-readable algorithm name (e.g., Perlin_Linear) and a compact integer representation suitable for network serialization and disk storage.

This component is fundamental to the protocol layer. By mapping to a fixed integer value, it ensures that world generation data is transmitted compactly and without ambiguity. The inclusion of a custom `fromValue` factory method, which throws a ProtocolException on invalid input, makes it a critical part of the data validation and deserialization pipeline. It prevents corrupted or malicious data from propagating further into the world generation systems.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. Developers do not and cannot manually instantiate this type.
- **Scope:** Application-wide static constants. The instances persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded by the JVM when the application terminates. There is no manual memory management required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a single, final integer field. The static VALUES array is also final and populated only once at class initialization. The state of a NoiseType constant can never change after creation.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature and the JVM's guarantees for enum initialization, NoiseType can be safely accessed and used by any number of threads concurrently without requiring external synchronization or locks.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value assigned to the enum constant for serialization. |
| fromValue(int value) | NoiseType | O(1) | **Static.** Deserializes an integer into its corresponding NoiseType. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
NoiseType is used to specify an algorithm in world generation logic or to deserialize incoming data from the network or a save file.

**Deserializing from a data stream:**
```java
// Reading an integer 'typeId' from a network buffer or file
int typeId = buffer.readInt();

// Safely convert the integer to a NoiseType
try {
    NoiseType noise = NoiseType.fromValue(typeId);
    worldGenerator.setNoise(noise);
} catch (ProtocolException e) {
    // Handle corrupted data, e.g., disconnect client or log error
    log.error("Received invalid NoiseType ID: " + typeId);
}
```

**Direct usage in code:**
```java
// Using the enum directly for logic branching
if (generator.getActiveNoiseType() == NoiseType.Perlin_Quintic) {
    // Apply specific logic for high-quality Perlin noise
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined to create a stable contract. Reordering the enum constants in the source file would break any system that relies on `ordinal()`, whereas the `getValue()` method is stable.
- **Unchecked Deserialization:** Never call `fromValue` with data from an untrusted source without a try-catch block. A failure to handle the potential ProtocolException can lead to an unhandled exception that crashes the server or client thread.

## Data Pipeline
NoiseType acts as a data model that is serialized for transport and deserialized upon receipt. It does not process data itself but ensures the integrity of data passing through the system.

**Serialization Flow:**
> Logic Layer -> **NoiseType.Perlin_Linear** -> `getValue()` -> `int 3` -> Network Buffer / File

**Deserialization Flow:**
> Network Buffer / File -> `int 3` -> `NoiseType.fromValue(3)` -> **NoiseType.Perlin_Linear** -> Logic Layer

