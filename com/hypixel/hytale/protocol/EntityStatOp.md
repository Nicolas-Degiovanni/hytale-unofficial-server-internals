---
description: Architectural reference for EntityStatOp
---

# EntityStatOp

**Package:** com.hypixel.hytale.protocol
**Type:** Utility Enum

## Definition
```java
// Signature
public enum EntityStatOp {
```

## Architecture & Concepts
EntityStatOp is a type-safe enumeration that defines the complete set of operations for modifying an entity's statistical attributes, such as health, speed, or attack power. It serves as a fundamental contract within the network protocol, ensuring that both the client and server interpret entity state changes consistently.

This enum is a critical component for network serialization. Each operation is mapped to a fixed integer value, which is the representation sent over the wire. This design minimizes packet size compared to sending string-based identifiers, optimizing for network performance. The class provides a bidirectional mapping: from the enum constant to its integer value for serialization, and from a received integer back to the corresponding enum constant for deserialization.

Its primary role is to decouple the network layer from the game logic. The network decoder uses EntityStatOp to parse incoming data into a structured, understandable command that the entity component system can then execute.

### Lifecycle & Ownership
- **Creation:** All enum constants (Init, Remove, Add, etc.) are instantiated automatically by the Java Virtual Machine (JVM) during class loading. This is a one-time, eager initialization process.
- **Scope:** Application-wide. Once the EntityStatOp class is loaded, its instances are available for the entire lifetime of the application. They are effectively permanent, global constants.
- **Destruction:** The enum constants are garbage collected only when the JVM shuts down. There is no manual memory management or destruction.

## Internal State & Concurrency
- **State:** EntityStatOp is **immutable**. Each enum constant holds a final integer field, which cannot be changed after instantiation. The static VALUES array is also effectively immutable after its one-time initialization.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability and the JVM's guarantees for enum initialization, it can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant, used for network serialization. |
| fromValue(int value) | EntityStatOp | O(1) | **Static Factory.** Converts a raw integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The most common use case is within a network packet handler or deserializer, where an integer is read from a byte buffer and converted into a high-level operation to be processed by the game logic.

```java
// In a packet deserializer
int operationId = buffer.readVarInt();
EntityStatOp operation = EntityStatOp.fromValue(operationId);

// In a game logic system
switch (operation) {
    case Add:
        entity.getHealth().add(value);
        break;
    case Set:
        entity.getHealth().set(value);
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Unsafe Deserialization:** Never call fromValue without a try-catch block or equivalent error handling in a production network handler. A malicious or corrupted packet could send an invalid integer, causing an unhandled ProtocolException that could disconnect the client.
- **Comparison by Value:** Do not compare operations using their integer values (e.g., `if (op.getValue() == 4)`). This defeats the purpose of a type-safe enum and creates brittle code. Always compare the enum instances directly (e.g., `if (op == EntityStatOp.Add)`).

## Data Pipeline
EntityStatOp acts as a translation point, converting a low-level network primitive into a high-level, domain-specific command.

> Flow:
> Network Byte Stream -> Protocol Decoder reads `int` -> `EntityStatOp.fromValue()` -> **EntityStatOp instance** -> Entity Component System -> Entity Attribute Update

