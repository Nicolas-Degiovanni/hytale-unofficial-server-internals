---
description: Architectural reference for BuilderToolArgType
---

# BuilderToolArgType

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum BuilderToolArgType {
```

## Architecture & Concepts
BuilderToolArgType is a static enumeration that defines the data type contract for arguments used in the builder tools network protocol. It functions as a type discriminator, allowing the client and server to unambiguously interpret the binary data that follows in a packet stream.

When a builder tool operation is serialized for network transmission, one of these enum's integer values is written to the buffer first. This integer tag informs the receiving end whether the subsequent bytes represent a boolean, a floating-point number, a block identifier, or another defined type. This class is therefore fundamental to the structural integrity and forward-compatibility of the builder tools system, acting as a rigid schema for inter-process communication.

The use of an explicit integer value for each constant, rather than relying on the default enum ordinal, is a deliberate design choice. It ensures that the protocol remains stable even if the declaration order of the enum constants changes in future versions of the source code.

## Lifecycle & Ownership
- **Creation:** All enum constants (Bool, Float, Int, etc.) are instantiated by the Java Virtual Machine during class loading. This process is guaranteed to happen only once.
- **Scope:** Application-wide. Once the BuilderToolArgType class is loaded, its instances persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum instances are reclaimed by the garbage collector only when the application's class loader is itself unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a private final integer, which is assigned at creation and can never be modified. The static VALUES array is also populated once during static initialization and is not subsequently changed.
- **Thread Safety:** This class is inherently thread-safe. The JVM guarantees that enum initialization is a safe, non-racy process. Its immutable nature means it can be safely read and passed between any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable, protocol-defined integer value for the enum constant. |
| fromValue(int value) | BuilderToolArgType | O(1) | Converts a raw integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet serialization and deserialization. The `fromValue` method is critical for interpreting incoming data from the network.

```java
// Example: Deserializing a builder tool argument from a network buffer
int typeId = buffer.readVarInt();
BuilderToolArgType argType = BuilderToolArgType.fromValue(typeId);

switch (argType) {
    case Bool:
        boolean value = buffer.readBoolean();
        // ... process boolean argument
        break;
    case Block:
        int blockId = buffer.readVarInt();
        // ... process block argument
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Never use the built-in `ordinal()` method for serialization. The protocol relies on the explicit integer from `getValue()`. Using `ordinal()` would create a brittle system that breaks if enum constants are reordered.
- **Ignoring Exceptions:** The `fromValue` method will throw a ProtocolException for unknown integer types. This is a critical failure condition indicating data corruption or a version mismatch. This exception must be caught and handled to prevent connection termination or undefined behavior. Do not wrap it in an empty catch block.
- **Comparison by Value:** Do not compare enum types by their integer value (e.g., `if (type.getValue() == 4)`). Always compare the enum instances directly for type safety and readability (e.g., `if (type == BuilderToolArgType.Block)`).

## Data Pipeline
This enum acts as a deserialization router, converting a low-level type identifier into a high-level, type-safe object that dictates subsequent processing logic.

> Flow:
> Network Byte Stream -> Read Integer (Type ID) -> **BuilderToolArgType.fromValue(typeId)** -> Switch Statement -> Typed Deserializer Logic

