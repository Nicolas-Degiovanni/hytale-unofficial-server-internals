---
description: Architectural reference for ChangeStatBehaviour
---

# ChangeStatBehaviour

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Enum

## Definition
```java
// Signature
public enum ChangeStatBehaviour {
   Add(0),
   Set(1);
```

## Architecture & Concepts
ChangeStatBehaviour is a type-safe enumeration that defines the specific operation to be performed on a player or entity statistic. It serves as a critical component of the network protocol's data contract, ensuring that both the client and server have an unambiguous understanding of how a stat value should be modified.

This enum acts as the deserialized, in-memory representation of a low-level integer value sent over the network. Its primary architectural function is to decouple game logic from the raw protocol data. Instead of passing magic numbers (0 for Add, 1 for Set) through the system, game logic can operate on the expressive and type-safe ChangeStatBehaviour constants.

The static factory method, fromValue, is the designated entry point for converting raw network data into a valid object, providing centralized validation and error handling for the protocol.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during class loading. They exist for the entire lifetime of the application.
- **Scope:** Application-wide. These constants are static and globally accessible.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual destruction or garbage collection of the constants themselves.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant has a final *value* field that is set at compile time and cannot be changed. The VALUES array is also static and final.
- **Thread Safety:** This class is inherently thread-safe. As an immutable enum, its instances can be safely shared and read across any number of threads without requiring synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value associated with the enum constant, used for serialization. |
| fromValue(int value) | ChangeStatBehaviour | O(1) | **Deserialization Entry Point.** Converts a raw integer from a network packet into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet deserialization to interpret a command field.

```java
// Reading from a hypothetical network buffer
int behaviourId = buffer.readVarInt();
ChangeStatBehaviour behaviour = ChangeStatBehaviour.fromValue(behaviourId);

// Game logic now uses the type-safe enum
switch (behaviour) {
    case Add:
        player.getStats().add(statId, value);
        break;
    case Set:
        player.getStats().set(statId, value);
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Value:** Do not compare enum instances by their integer value. The JVM guarantees that there is only one instance of each enum constant, so direct object comparison is safer and more performant.
    - **BAD:** `if (behaviour.getValue() == 0)`
    - **GOOD:** `if (behaviour == ChangeStatBehaviour.Add)`
- **Reliance on Ordinal:** Never use the built-in `ordinal()` method for serialization or business logic. The explicit `value` field is the canonical network representation. The order of enum declarations can change, which would break any logic based on `ordinal()`.

## Data Pipeline
ChangeStatBehaviour is a key transformation step in the data pipeline for entity statistic updates, converting a raw network primitive into a high-level command.

> Flow:
> Network Packet (containing raw int) -> Protocol Decoder -> **ChangeStatBehaviour.fromValue(intValue)** -> StatUpdateEvent -> Game Logic System

