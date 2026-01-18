---
description: Architectural reference for DrawType
---

# DrawType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum DrawType {
```

## Architecture & Concepts
The DrawType enum provides a type-safe, compile-time constant representation for entity and object rendering styles within the Hytale engine. Its primary function is to serve as a contract for network communication, translating between a compact integer representation on the wire and a descriptive, readable type in the game client and server code.

Located in the core protocol package, this enum is fundamental to the serialization and deserialization of world data. It eliminates the use of "magic numbers" for rendering instructions, thereby improving code clarity and reducing the risk of errors. When a client receives data about an entity, the integer draw type is converted via the static fromValue method into a DrawType instance, which the rendering pipeline then uses to select the appropriate shader, model, or primitive drawing routine.

The inclusion of a static VALUES array is a performance optimization. It pre-caches the result of the expensive values() method call, which would otherwise create a new array on each invocation. This provides fast, zero-allocation lookups in the fromValue method, critical for high-performance network packet processing.

## Lifecycle & Ownership
- **Creation:** All instances (Empty, GizmoCube, etc.) are instantiated by the Java Virtual Machine during class loading. They are static, final constants managed entirely by the JVM.
- **Scope:** Application-scoped. The enum constants exist for the entire lifetime of the application process.
- **Destruction:** The instances are reclaimed only when the application's class loader is garbage collected, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The internal integer value for each constant is assigned at creation and cannot be changed.
- **Thread Safety:** This class is fully thread-safe. As immutable, globally unique constants, instances of DrawType can be safely passed between and accessed by any thread without requiring synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value associated with the enum constant, intended for serialization. |
| fromValue(int value) | static DrawType | O(1) | Deserializes an integer from a network stream into its corresponding DrawType. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the decoding of network packets to determine how to render an object received from the server.

```java
// Example: Processing a render instruction from a network buffer
int rawDrawType = buffer.readVarInt();
DrawType drawType = DrawType.fromValue(rawDrawType);

switch (drawType) {
    case Cube:
        renderingEngine.drawVoxelCube(entity);
        break;
    case Model:
        renderingEngine.drawEntityModel(entity);
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Never use the built-in `ordinal()` method for serialization. The integer value of an ordinal can change if the enum declaration order is modified, which would break network compatibility between different client and server versions. Always use the explicit `getValue()` method.
- **Reflection:** Do not attempt to create new instances of this enum using reflection. This violates the core guarantees of the Java enum type and can lead to severe, unpredictable application instability.

## Data Pipeline
DrawType acts as a deserialization gateway, converting a low-level network primitive into a high-level, type-safe instruction for the rendering system.

> Flow:
> Network Byte Stream -> Protocol Decoder -> `int` value -> **DrawType.fromValue(value)** -> Rendering System Logic

