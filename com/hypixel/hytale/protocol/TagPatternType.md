---
description: Architectural reference for TagPatternType
---

# TagPatternType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum TagPatternType {
```

## Architecture & Concepts
The TagPatternType enum defines a fixed set of logical operators used for constructing and evaluating tag-based queries. It serves as a type-safe representation of operators that can be applied to game data, such as filtering items, selecting entities, or matching content definitions.

This component is fundamental to the protocol layer, acting as a translation mechanism between a low-level, serializable integer and a high-level, self-documenting logical concept. By encoding operators like AND and OR as simple integers, the system minimizes network bandwidth and storage footprint. The static `fromValue` method provides a robust deserialization entry point, ensuring that only valid operator codes can be processed by the engine, thereby preventing data corruption.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the Java Virtual Machine during class loading. They are effectively singletons managed by the runtime.
- **Scope:** Application-wide. An instance of each pattern type exists for the entire lifetime of the application.
- **Destruction:** The enum and its constants are garbage collected only when the application's ClassLoader is unloaded, which typically occurs at JVM shutdown. No manual memory management is required.

## Internal State & Concurrency
- **State:** Deeply **Immutable**. Each enum constant holds a final primitive integer. Its state cannot be modified after creation. The static VALUES array is populated once during static initialization and is not intended for modification.
- **Thread Safety:** Inherently **thread-safe**. As immutable singletons, instances of TagPatternType can be safely shared and read across multiple threads without any synchronization mechanisms. All methods are pure functions or simple field accessors.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the primitive integer value associated with the enum constant, used for serialization. |
| fromValue(int value) | static TagPatternType | O(1) | Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves deserializing an integer from a network stream or data file and converting it into a type-safe TagPatternType object for use in game logic, typically within a switch statement.

```java
// Example: Reading an operator from a network buffer
int operatorCode = buffer.readVarInt();
TagPatternType pattern = TagPatternType.fromValue(operatorCode);

switch (pattern) {
    case And:
        // handle logical AND
        break;
    case Or:
        // handle logical OR
        break;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Unsafe Casting:** Do not attempt to bypass the `fromValue` method. Relying on `TagPatternType.VALUES[value]` directly circumvents the critical bounds checking and can lead to an ArrayIndexOutOfBoundsException if the data is malformed.
- **Reflection:** Do not attempt to create new instances of this enum using reflection. This violates the singleton guarantee of enums and can lead to unpredictable behavior and state corruption throughout the system.

## Data Pipeline
TagPatternType is not a processing stage itself but rather a data token that flows through various systems. It provides semantic meaning to a raw integer value within a larger data stream.

> Flow:
> Network Byte Stream -> Protocol Decoder -> Raw Integer -> **TagPatternType.fromValue()** -> Query Execution Engine -> Filtered Game State

