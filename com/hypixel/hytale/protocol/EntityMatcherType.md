---
description: Architectural reference for EntityMatcherType
---

# EntityMatcherType

**Package:** com.hypixel.hytale.protocol
**Type:** Enum Constant / Utility

## Definition
```java
// Signature
public enum EntityMatcherType {
```

## Architecture & Concepts
The EntityMatcherType enum provides a type-safe, integer-backed representation for different entity matching strategies within the network protocol. It serves as a foundational data model, translating between raw integer identifiers used in network packets and strongly-typed, self-documenting constants used within the game engine.

This class is a critical component of the protocol's serialization and deserialization layer. Its primary function is to eliminate the use of "magic numbers" when identifying entity types, thereby improving code readability, maintainability, and safety. The `fromValue` static factory method is the designated entry point for converting incoming network data into a valid game state object, while `getValue` is used for the reverse process of serialization.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during the class loading phase. This is a one-time, static initialization that occurs before any application code can access the type.
- **Scope:** Application-wide. The instances (Server, VulnerableMatcher, Player) are static and persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded only when the defining ClassLoader is garbage collected, which typically coincides with application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that is assigned at creation and cannot be modified. The static `VALUES` array is a performance optimization for the `fromValue` method, caching the result of the `values()` call to prevent repeated array allocation.
- **Thread Safety:** Inherently thread-safe. As an immutable type with pre-initialized, static instances, EntityMatcherType can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the protocol-defined integer identifier for the enum constant. |
| fromValue(int value) | EntityMatcherType | O(1) | **[Deserialization]** Looks up an enum constant from its integer identifier. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets that reference a specific type of entity.

**Deserialization (Reading from a packet):**
```java
// Read an integer from the network buffer
int matcherId = buffer.readVarInt();

// Convert the raw integer into a safe, usable type
EntityMatcherType type = EntityMatcherType.fromValue(matcherId);

// Use the type in game logic
if (type == EntityMatcherType.Player) {
    // Handle player-specific logic
}
```

**Serialization (Writing to a packet):**
```java
// Get the integer value for a known type
EntityMatcherType typeToWrite = EntityMatcherType.Server;
int id = typeToWrite.getValue();

// Write the integer to the network buffer
buffer.writeVarInt(id);
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined and decoupled from declaration order. Relying on `ordinal()` is brittle and will lead to protocol mismatches if the enum declaration order is ever changed. Always use `getValue()` and `fromValue()`.
- **Magic Numbers:** Avoid comparing against raw integer values in application logic. The purpose of this enum is to provide named constants.

    **BAD:**
    `if (entity.getMatcherId() == 2) { ... }`

    **GOOD:**
    `if (entity.getMatcherType() == EntityMatcherType.Player) { ... }`

## Data Pipeline
EntityMatcherType acts as a data model that is serialized into and deserialized from the network data stream. It does not process data itself but represents a state within the data.

> **Serialization Flow:**
> Game State (`EntityMatcherType.Player`) -> `getValue()` -> Integer (`2`) -> Protocol Buffer -> Network Socket

> **Deserialization Flow:**
> Network Socket -> Protocol Buffer -> Integer (`2`) -> `fromValue(2)` -> Game State (`EntityMatcherType.Player`)

