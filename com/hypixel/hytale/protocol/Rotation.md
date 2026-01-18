---
description: Architectural reference for Rotation
---

# Rotation

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum Rotation {
```

## Architecture & Concepts
The Rotation enum is a fundamental value type within the Hytale network protocol layer. It provides a constrained, type-safe representation for cardinal, 90-degree rotations of objects and blocks within the game world.

Its primary architectural role is to enforce data integrity during serialization and deserialization of game state. By mapping specific integer values (0-3) to named constants, it eliminates the possibility of "magic numbers" and invalid orientation states in network packets or save files. This class acts as a contract between the client and server, ensuring that both systems interpret object orientation identically.

It is not a service or a manager, but rather a core building block used within larger data structures like entity state packets or block placement commands.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (None, Ninety, OneEighty, TwoSeventy) are created and initialized by the Java Virtual Machine (JVM) during class loading. They are compile-time constants.
- **Scope:** As static final instances, all Rotation constants exist for the entire lifetime of the application. They are effectively global singletons managed by the JVM.
- **Destruction:** The instances are garbage collected only when the application's ClassLoader is unloaded, which typically happens at application shutdown. User code cannot and should not attempt to manage their lifecycle.

## Internal State & Concurrency
- **State:** The Rotation enum is **immutable**. Each instance holds a final primitive integer, *value*, which is set at creation time and can never be changed. The static VALUES array is also effectively immutable.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutability guarantees that it can be safely accessed, passed, and read from any number of concurrent threads without the need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation used for network serialization. |
| fromValue(int value) | Rotation | O(1) | **[Factory Method]** Deserializes an integer into a Rotation instance. Throws ProtocolException if the value is out of the valid range [0, 3]. |
| VALUES | Rotation[] | O(1) | A cached, public array of all enum constants. Primarily for high-performance iteration or lookups. |

## Integration Patterns

### Standard Usage
The primary use case is converting between the integer representation from a network stream and the type-safe enum object. Error handling is critical when deserializing.

```java
// Deserializing a rotation value from a network packet
int rotationValue = packet.readVarInt();
try {
    Rotation blockRotation = Rotation.fromValue(rotationValue);
    world.setBlockRotation(position, blockRotation);
} catch (ProtocolException e) {
    // Handle malformed packet: log the error, disconnect the client, etc.
    log.error("Received invalid rotation value: " + rotationValue, e);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The explicit `value` field is the canonical representation. Relying on `ordinal()` is brittle and can break if the enum declaration order changes.
- **Ignoring Exceptions:** Never call `fromValue` without a try-catch block when processing external data. An unhandled ProtocolException can crash a network thread or lead to an inconsistent game state.
- **Unsafe Casting:** Do not attempt to bypass the `fromValue` factory by casting an integer. This will fail and indicates a misunderstanding of the type system.

## Data Pipeline
The Rotation enum serves as a translation point between raw network data and the structured in-memory game state.

> **Serialization Flow:**
> Game Object State (Rotation.Ninety) -> `getValue()` -> Integer (1) -> Protocol Buffer -> Network Byte Stream

> **Deserialization Flow:**
> Network Byte Stream -> Protocol Parser -> Integer (1) -> **`Rotation.fromValue(1)`** -> Game Object State (Rotation.Ninety)

