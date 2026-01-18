---
description: Architectural reference for HorizontalSelectorDirection
---

# HorizontalSelectorDirection

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum / Value Object

## Definition
```java
// Signature
public enum HorizontalSelectorDirection {
```

## Architecture & Concepts
HorizontalSelectorDirection is a type-safe enumeration that represents a discrete, two-state direction within the Hytale network protocol. Its primary function is to provide a robust and explicit representation for horizontal movement or selection, replacing ambiguous integer flags or "magic numbers".

This enum is a fundamental component of the protocol layer, acting as a contract between the client and server. It ensures that data representing horizontal direction is serialized to a compact integer format for network transmission and deserialized back into a strongly-typed object on the receiving end. The use of an enum prevents a class of bugs related to invalid or out-of-range integer values, as any attempt to deserialize an undefined value will result in a controlled ProtocolException.

## Lifecycle & Ownership
-   **Creation:** The enum constants, ToLeft and ToRight, are instantiated by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs once.
-   **Scope:** The instances are static, final, and globally accessible for the entire lifetime of the application. They are effectively application-scoped singletons.
-   **Destruction:** The enum instances are reclaimed by the JVM only when the application shuts down. There is no manual memory management required.

## Internal State & Concurrency
-   **State:** HorizontalSelectorDirection is **immutable**. Each enum constant holds a final integer value that is assigned at compile time and cannot be changed. The static VALUES array is also final.
-   **Thread Safety:** This class is inherently **thread-safe**. As an immutable, globally-accessible type, it can be safely read from any number of concurrent threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for network serialization. |
| fromValue(int value) | static HorizontalSelectorDirection | O(1) | Deserializes an integer from a network stream into an enum constant. Throws ProtocolException if the value is out of bounds. |
| VALUES | static HorizontalSelectorDirection[] | O(1) | A cached, public array of all defined enum constants. **Warning:** Direct modification of this array via reflection will lead to undefined behavior. |

## Integration Patterns

### Standard Usage
This enum should be used within data transfer objects (DTOs) or network packet definitions to represent a choice between left and right. The serialization and deserialization logic should use the provided static methods.

```java
// Example: Deserializing a value from a network buffer
int directionValue = networkBuffer.readVarInt();
HorizontalSelectorDirection direction = HorizontalSelectorDirection.fromValue(directionValue);

// Example: Using the value in game logic
if (direction == HorizontalSelectorDirection.ToLeft) {
    // handle left selection
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Magic Numbers:** Do not use the raw integer values 0 or 1 in business logic. This defeats the purpose of type safety and creates brittle code that is difficult to refactor. Always compare against the enum constants directly.
    ```java
    // BAD: Prone to errors and hard to read
    if (packet.getDirectionValue() == 0) { ... }

    // GOOD: Type-safe and self-documenting
    if (packet.getDirection() == HorizontalSelectorDirection.ToLeft) { ... }
    ```
-   **Bypassing Deserialization:** Do not manually check the integer value before calling fromValue. The method is designed to be the single source of truth for validation and will throw a specific, catchable exception for invalid data.

## Data Pipeline
HorizontalSelectorDirection is a critical component in the network serialization and deserialization pipeline.

> **Outbound Flow (Serialization):**
> Game Logic State -> Packet Object holding **HorizontalSelectorDirection.ToRight** -> Serializer calls `getValue()` -> Network Stream receives integer `1`

> **Inbound Flow (Deserialization):**
> Network Stream provides integer `1` -> Deserializer calls `fromValue(1)` -> Packet Object holding **HorizontalSelectorDirection.ToRight** -> Game Logic receives typed object

