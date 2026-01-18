---
description: Architectural reference for InteractionTarget
---

# InteractionTarget

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum InteractionTarget {
```

## Architecture & Concepts
InteractionTarget is a type-safe enumeration that represents the intended recipient of a gameplay interaction within the network protocol. It serves as a fundamental building block for any packet that describes an action between entities, such as a player interacting with an object or another player.

The primary architectural function of this enum is to replace "magic numbers" in the network stream with a robust, self-documenting, and compile-time-checked constant. By mapping specific integer values (0, 1, 2) to named constants (User, Owner, Target), it provides a stable contract for serialization and deserialization, ensuring that both the client and server interpret interaction data consistently.

This component resides at the lowest level of the protocol definition layer and is considered a primitive data type within the Hytale network architecture.

### Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated by the Java Virtual Machine during the class-loading phase. They are not created dynamically by application code.
-   **Scope:** The defined instances (User, Owner, Target) are singletons that persist for the entire application lifetime.
-   **Destruction:** The constants are managed by the JVM and are only eligible for cleanup when the defining classloader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
-   **State:** Immutable. Each enum constant holds a final integer value defined at compile time. The class itself is stateless.
-   **Thread Safety:** Inherently thread-safe. As static, final instances created by the JVM, these constants can be safely accessed and passed between any number of threads without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the primitive integer value for serialization. |
| fromValue(int value) | InteractionTarget | O(1) | Deserializes an integer from the network stream into its corresponding enum constant. Throws ProtocolException if the value is out of the defined range. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets. The typical pattern involves reading an integer from a buffer and converting it to an InteractionTarget, or converting an InteractionTarget to an integer before writing it to a buffer.

```java
// Deserialization from a network buffer
int rawTargetValue = buffer.readVarInt();
InteractionTarget target = InteractionTarget.fromValue(rawTargetValue);

// Serialization to a network buffer
InteractionTarget myTarget = InteractionTarget.Owner;
buffer.writeVarInt(myTarget.getValue());
```

### Anti-Patterns (Do NOT do this)
-   **Numeric Comparison:** Do not compare enum instances using their underlying integer values. This defeats the purpose of type safety and makes the code brittle.

    ```java
    // BAD: Prone to errors if integer values change
    if (target.getValue() == 1) {
        // ...
    }

    // GOOD: Type-safe and self-documenting
    if (target == InteractionTarget.Owner) {
        // ...
    }
    ```
-   **Unhandled Deserialization:** Failure to handle the ProtocolException thrown by fromValue can lead to unrecoverable protocol state corruption. Always wrap calls in a try-catch block during packet parsing.

## Data Pipeline
InteractionTarget acts as a translation point between the raw binary representation on the network and the type-safe object model used by the game engine.

> **Deserialization Flow:**
> Network Packet (Integer) -> Protocol Buffer Reader -> **InteractionTarget.fromValue()** -> Game Logic (InteractionTarget Object)

> **Serialization Flow:**
> Game Logic (InteractionTarget Object) -> **InteractionTarget.getValue()** -> Protocol Buffer Writer -> Network Packet (Integer)

