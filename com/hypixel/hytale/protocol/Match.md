---
description: Architectural reference for Match
---

# Match

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum Match {
```

## Architecture & Concepts
The Match enum is a foundational component of the Hytale network protocol layer. It provides a type-safe representation for a constrained set of matching conditions, replacing the ambiguity and risk associated with "magic numbers" or raw integers in the data stream.

Its primary architectural role is to act as a data contract between the client and server. By defining a strict set of possible values (All, None), it ensures that both ends of a connection operate on a shared, validated understanding of a particular state. The inclusion of a serialization and deserialization mechanism (`getValue`, `fromValue`) makes it a self-contained unit for network I/O, responsible for both its representation and its validation.

The `fromValue` method is a critical security and stability gate. It actively prevents protocol desynchronization or malformed packets by throwing a `ProtocolException` if an undefined integer value is received, allowing the network layer to immediately reject invalid data.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during the class loading phase. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** As static final instances, Match constants exist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum and its constants are unloaded from memory only when the application's ClassLoader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** The Match enum is **immutable**. The internal integer `value` is a final field assigned at creation and cannot be modified. The static `VALUES` array is also final.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that no race conditions can occur when accessing its state from multiple threads. The static factory method `fromValue` is also safe as it is a pure function with no side effects or shared mutable state.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its network-protocol integer representation. |
| fromValue(int value) | Match | O(1) | Deserializes an integer from the network into a valid Match constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The `fromValue` method is the designated entry point when deserializing data from a network buffer. The `getValue` method is used for serialization.

```java
// How to deserialize a value from a network packet
try {
    int rawValue = packet.readInt();
    Match condition = Match.fromValue(rawValue);
    // Use the validated 'condition' in game logic
} catch (ProtocolException e) {
    // Handle protocol violation, e.g., disconnect the client
    connection.disconnect("Invalid Match value received.");
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined in the constructor to create a stable network contract. Relying on `ordinal()` is fragile and will break the protocol if the enum declaration order changes.
- **Ignoring Exceptions:** Swallowing the `ProtocolException` from `fromValue` is a severe error. This exception signifies a corrupt or malicious packet, and failure to act on it can lead to undefined behavior or security vulnerabilities. The connection should be terminated.
- **Magic Number Comparison:** Avoid comparing against the raw integer values in business logic. The purpose of the enum is to provide semantic meaning.

   **BAD:**
   `if (condition.getValue() == 0) { ... }`

   **GOOD:**
   `if (condition == Match.All) { ... }`

## Data Pipeline
The Match enum serves as a validation and translation point in the data ingress pipeline.

> Flow:
> Network Byte Stream -> Integer Deserializer -> **Match.fromValue(int)** -> Validated Game Logic State

