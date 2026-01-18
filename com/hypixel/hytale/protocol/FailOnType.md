---
description: Architectural reference for FailOnType
---

# FailOnType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Protocol Data Type

## Definition
```java
// Signature
public enum FailOnType {
```

## Architecture & Concepts
FailOnType is a type-safe enumeration that defines a set of behavioral flags for protocol operations, likely related to targeting or interaction logic. It translates a low-level integer value from the network stream into a high-level, self-documenting constant used by game logic.

This enum is a foundational element of the network protocol's data model. Its primary role is to eliminate "magic numbers" from the codebase, providing clear, readable constants like FailOnType.Entity instead of an ambiguous integer like 1.

The inclusion of a static `fromValue` factory method is a critical deserialization pattern. It acts as a validation gate, ensuring that only valid, defined integer values from a network packet are converted into an enum instance. Any undefined value results in a ProtocolException, causing the system to fail fast rather than proceed with corrupt or unpredictable state.

## Lifecycle & Ownership
- **Creation:** All instances (Neither, Entity, Block, Either) are created and initialized by the Java Virtual Machine during class loading. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** Application-wide static constants. They persist for the entire lifetime of the application.
- **Destruction:** Instances are reclaimed by the garbage collector only when the application's class loader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant is a singleton instance with a final `value` field. Its state cannot be modified after creation.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of FailOnType can be safely accessed, passed, and read by any number of threads without requiring locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation for serialization. |
| fromValue(int value) | FailOnType | O(1) | Deserializes an integer into a FailOnType instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet deserialization to convert a raw integer from the network buffer into a strongly-typed object.

```java
// How a developer should normally use this
int rawFlag = networkBuffer.readVarInt();
FailOnType condition = FailOnType.fromValue(rawFlag);

// ... later in game logic
if (condition == FailOnType.Entity) {
    // Handle entity-specific failure logic
}
```

### Anti-Patterns (Do NOT do this)
- **Numeric Comparison:** Do not use the integer value for logical comparisons. The `==` operator is guaranteed to work for enums and is more readable and type-safe.
  - **BAD:** `if (condition.getValue() == 1)`
  - **GOOD:** `if (condition == FailOnType.Entity)`
- **Uncached `values()` Call:** In performance-critical code, avoid calling `FailOnType.values()` repeatedly as it allocates a new array on each invocation. This class mitigates this by providing a pre-cached static `VALUES` array.
  - **BAD (in a loop):** `for (FailOnType type : FailOnType.values())`
  - **GOOD (in a loop):** `for (FailOnType type : FailOnType.VALUES)`

## Data Pipeline
FailOnType serves as a translation layer between the raw network stream and the game's internal logic.

**Serialization (Outgoing Data)**
> Flow:
> Game Logic State -> **FailOnType.Block** -> `getValue()` -> `Integer (2)` -> Network Buffer

**Deserialization (Incoming Data)**
> Flow:
> Network Buffer -> `Integer (2)` -> `fromValue(2)` -> **FailOnType.Block** -> Game Logic State

