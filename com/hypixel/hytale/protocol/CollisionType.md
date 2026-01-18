---
description: Architectural reference for CollisionType
---

# CollisionType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum CollisionType {
   Hard(0),
   Soft(1);
```

## Architecture & Concepts
The CollisionType enum provides a type-safe, compile-time constant representation for different physics collision behaviors within the game engine. Its primary architectural role is to serve as a standardized data model for serializing and deserializing entity collision properties across the network protocol.

By mapping a human-readable name (e.g., Hard) to a fixed integer value, it ensures that both the client and server have an unambiguous understanding of how an entity should interact with the game world. The use of an enum prevents invalid or out-of-range values from being used in game logic, enforcing data integrity at the type system level. It is a fundamental building block for the physics and networking layers.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during the class loading phase. There will only ever be one instance of CollisionType.Hard and one instance of CollisionType.Soft for the entire application lifecycle.
- **Scope:** Application-wide. These instances are static and globally accessible once the CollisionType class is loaded.
- **Destruction:** The enum instances are reclaimed by the garbage collector only when the application's ClassLoader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Immutable. The internal integer value of each enum constant is final and assigned at creation. The enum instances themselves are effectively singletons.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature and the JVM's handling of enum instantiation, CollisionType can be safely accessed and used across multiple threads without any external synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the enum constant, primarily for network serialization. |
| fromValue(int value) | CollisionType | O(1) | A static factory method that deserializes an integer from a network packet back into its corresponding enum constant. Throws ProtocolException if the integer is not a valid value. |

## Integration Patterns

### Standard Usage
This enum is most frequently used when deserializing network data. The fromValue method is the designated entry point for converting a raw integer from a packet into a safe game object.

```java
// Reading a collision type from a network buffer
int rawCollisionValue = buffer.readVarInt();
CollisionType type = CollisionType.fromValue(rawCollisionValue);

// Using the type in game logic
if (type == CollisionType.Hard) {
    // Execute hard collision physics
}
```

### Anti-Patterns (Do NOT do this)
- **Magic Numbers:** Avoid using the raw integer values directly in game logic. Rely on the enum constants for clarity, type safety, and maintainability.

```java
// BAD: Prone to errors and hard to read
if (entity.getCollisionValue() == 0) { ... }

// GOOD: Type-safe and self-documenting
if (entity.getCollisionType() == CollisionType.Hard) { ... }
```

- **Manual Value Checking:** Do not manually check if a value is valid before calling fromValue. The method is designed to perform this validation and throw a standardized exception.

## Data Pipeline
CollisionType acts as a data contract within the network pipeline. It is not an active processing component but rather the data that is being processed.

> **Serialization Flow:**
> Game Entity State -> **CollisionType.Hard** -> getValue() -> 0 (int) -> Network Buffer -> Server/Client

> **Deserialization Flow:**
> Server/Client -> Network Buffer -> 0 (int) -> fromValue(0) -> **CollisionType.Hard** -> Game Entity State

