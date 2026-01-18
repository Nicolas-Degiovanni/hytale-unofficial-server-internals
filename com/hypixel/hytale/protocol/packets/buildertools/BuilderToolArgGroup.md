---
description: Architectural reference for BuilderToolArgGroup
---

# BuilderToolArgGroup

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Type-Safe Enum

## Definition
```java
// Simplified Signature
public enum BuilderToolArgGroup {
   Tool(0),
   Brush(1);
}
```

## Architecture & Concepts
BuilderToolArgGroup is a type-safe enumeration that represents distinct categories of arguments within the builder tools network protocol. Its primary role is to act as a serialization and deserialization bridge, converting low-level integer identifiers from the network stream into high-level, self-documenting constants used by the game logic.

This enum is a fundamental component of the protocol layer, ensuring that data related to builder tool commands is unambiguous and robust. By mapping integer values to named constants like Tool and Brush, it eliminates the use of "magic numbers", thereby improving code clarity and reducing the risk of errors during protocol evolution. The inclusion of a custom ProtocolException in its deserialization logic provides a standardized error-handling mechanism for malformed or unexpected packet data.

### Lifecycle & Ownership
- **Creation:** Enum instances are created and initialized by the Java Virtual Machine during class loading. These instances are compile-time constants.
- **Scope:** Application-level. The enum constants persist for the entire lifetime of the application.
- **Destruction:** Instances are managed by the JVM and are reclaimed only upon application shutdown. There is no manual memory management.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that is assigned at creation and cannot be modified. The static VALUES array is a performance cache that is also effectively immutable after its one-time initialization.
- **Thread Safety:** This class is inherently thread-safe. As a collection of immutable singletons, its instances and static methods can be safely accessed from any thread without requiring external synchronization or locks.

## API Surface
The public contract is designed for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value associated with the enum constant, suitable for serialization into a network packet. |
| fromValue(int value) | BuilderToolArgGroup | O(1) | **Static Factory.** Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the packet decoding process to translate a raw integer from a byte buffer into a strongly-typed object.

```java
// Example within a hypothetical packet decoder
int groupId = buffer.readVarInt();
BuilderToolArgGroup argGroup = BuilderToolArgGroup.fromValue(groupId);

// Game logic can now use the type-safe enum
switch (argGroup) {
    case Tool:
        // handle tool-specific arguments
        break;
    case Brush:
        // handle brush-specific arguments
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinal:** Do not use the built-in `ordinal()` method for serialization. The protocol relies on the explicit integer `value` field. The order of declaration may change, but the assigned value is a stable contract.
- **Manual Value Comparison:** Avoid comparing the raw integer value in game logic. The purpose of the enum is to provide type safety.

    ```java
    // BAD: Brittle and defeats the purpose of the enum
    if (argGroup.getValue() == 0) { ... }

    // GOOD: Robust and readable
    if (argGroup == BuilderToolArgGroup.Tool) { ... }
    ```
- **Ignoring Exceptions:** The `fromValue` method can throw a ProtocolException. This is a critical signal of a malformed packet or a protocol version mismatch. It must not be ignored.

## Data Pipeline
BuilderToolArgGroup serves as a validation and translation step in the inbound data pipeline for builder tool packets.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> **BuilderToolArgGroup.fromValue(intValue)** -> Typed BuilderToolArgGroup Instance -> Game Command Logic

