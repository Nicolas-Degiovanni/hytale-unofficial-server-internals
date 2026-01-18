---
description: Architectural reference for EntityStatResetBehavior
---

# EntityStatResetBehavior

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum EntityStatResetBehavior {
```

## Architecture & Concepts
The EntityStatResetBehavior enum defines a strict, type-safe contract for how an entity's statistical value, such as health or mana, should behave when reset. It is a fundamental component of the network protocol layer, designed to eliminate ambiguity and prevent "magic number" bugs that can arise from using raw integers to represent state.

This enum serves two primary purposes:
1.  **Semantic Clarity:** It provides self-documenting, readable constants (InitialValue, MaxValue) for use in game logic.
2.  **Serialization Contract:** It defines a stable integer representation for each behavior, ensuring that client and server can reliably serialize and deserialize this state for network transmission.

Its existence is critical for maintaining protocol compatibility and robustness between different versions of the client and server.

### Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated once by the JVM during the class loading phase. They are effectively compile-time constants managed by the Java runtime.
-   **Scope:** Application-wide singleton. An instance like EntityStatResetBehavior.MaxValue exists for the entire lifetime of the application.
-   **Destruction:** The constants are reclaimed by the garbage collector only when the defining class loader is unloaded, which typically occurs at JVM shutdown. There is no manual lifecycle management.

## Internal State & Concurrency
-   **State:** Deeply **immutable**. The internal integer value is a final field set by the private constructor during class loading. The set of enum instances is fixed and cannot be modified at runtime.
-   **Thread Safety:** Inherently **thread-safe**. As immutable singletons, instances of EntityStatResetBehavior can be safely accessed and shared across any number of threads without requiring synchronization primitives. The static VALUES array is also populated safely during the JVM's class initialization process.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value associated with the enum constant, intended for serialization. |
| fromValue(int value) | EntityStatResetBehavior | O(1) | A static factory method that deserializes an integer into its corresponding enum constant. Throws ProtocolException if the integer is not a valid representation. |

## Integration Patterns

### Standard Usage
This enum is primarily used by network packet encoders and decoders to process game state. Game logic should use the enum constants directly for comparisons and assignments.

```java
// Example: Decoding a stat reset instruction from a network packet
int behaviorId = packet.readVarInt();
EntityStatResetBehavior behavior = EntityStatResetBehavior.fromValue(behaviorId);

// Example: Applying the behavior in game logic
if (behavior == EntityStatResetBehavior.MaxValue) {
    player.setHealth(player.getMaxHealth());
}

// Example: Encoding a packet to send to the client
packet.writeVarInt(EntityStatResetBehavior.InitialValue.getValue());
```

### Anti-Patterns (Do NOT do this)
-   **Using Ordinal for Serialization:** Do not use the built-in `ordinal()` method for serialization. The integer value defined in the constructor and retrieved via `getValue()` is the canonical network representation. Relying on `ordinal()` is extremely brittle, as reordering the enum declarations in the source file will break network compatibility.
-   **Ignoring Exceptions:** The `fromValue` method will throw a ProtocolException for any invalid integer. This is a critical validation step. Do not wrap calls in an empty catch block, as this can hide protocol desynchronization errors.
-   **Manual Comparison:** Avoid comparing the integer values directly in game logic. Use the enum constants for all comparisons to maintain readability and type safety.

## Data Pipeline
EntityStatResetBehavior is not a processing component but rather a data model that flows through the network and game logic pipelines. It represents a piece of deserialized state.

> **Flow:**
> Network Byte Stream -> Packet Decoder -> `EntityStatResetBehavior.fromValue(int)` -> **EntityStatResetBehavior Instance** -> Game Logic System -> Entity State Update

