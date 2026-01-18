---
description: Architectural reference for ValueType
---

# ValueType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum ValueType {
```

## Architecture & Concepts
The ValueType enum is a fundamental component of the Hytale protocol layer, designed to provide a type-safe and network-efficient representation for interpreting numerical values. It standardizes how the system distinguishes between absolute quantities (e.g., +10 health) and relative percentages (e.g., +10% of max health).

This enum is critical for serializing game mechanics data. By mapping each type to a compact integer, it minimizes packet size. The static factory method, fromValue, acts as the deserialization gateway, ensuring that only valid integer representations are converted back into enum constants, thereby preventing data corruption from malformed or malicious packets. Its primary role is to eliminate "magic numbers" (e.g., using 0 for percent, 1 for absolute) in the application logic, replacing them with self-documenting, compile-time-checked constants.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during the class-loading phase. They are effectively static singletons created before any application code runs.
- **Scope:** Application-wide. A ValueType constant persists for the entire lifetime of the JVM process.
- **Destruction:** The constants are reclaimed only when the application's ClassLoader is garbage collected, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Immutable. The internal integer value of each enum constant is final and assigned at compile time. The state of a ValueType instance can never be modified post-creation.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, ValueType can be safely accessed, passed, and read from any thread without requiring external synchronization or locks. The static VALUES array is populated during class initialization and is not modified thereafter, making it safe for concurrent reads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation used for network serialization. |
| fromValue(int value) | ValueType | O(1) | Deserializes an integer into a ValueType. Throws ProtocolException if the integer is out of bounds. |

## Integration Patterns

### Standard Usage
ValueType should be used within data transfer objects (DTOs) or game logic to define how a corresponding numerical field is interpreted.

```java
// Example: Applying a stat modifier based on its type
public void applyStatModifier(float amount, ValueType type) {
    if (type == ValueType.Absolute) {
        this.stat += amount;
    } else if (type == ValueType.Percent) {
        this.stat += this.baseStat * (amount / 100.0f);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never use the raw integer values in application logic. The integer representation is an implementation detail for serialization and should not leak into other layers.
    ```java
    // BAD: Hardcoding the integer value
    if (modifierType == 1) { /* ... */ }

    // GOOD: Using the enum constant
    if (modifierType == ValueType.Absolute.getValue()) { /* ... */ } // Still bad
    if (type == ValueType.Absolute) { /* ... */ } // Correct
    ```
- **Ignoring Deserialization Failures:** The fromValue method can throw a ProtocolException. Network-facing code must handle this exception to prevent crashes from invalid data.

## Data Pipeline
ValueType is not a processing component but rather a data model that flows through serialization and deserialization pipelines.

> Flow:
> Game State (ValueType.Percent) -> Protocol Serializer (calls getValue) -> Network Packet (contains integer `0`) -> Protocol Deserializer (calls fromValue) -> Reconstructed Game State (ValueType.Percent)

