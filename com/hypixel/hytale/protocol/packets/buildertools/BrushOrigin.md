---
description: Architectural reference for BrushOrigin
---

# BrushOrigin

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Type

## Definition
```java
// Signature
public enum BrushOrigin {
```

## Architecture & Concepts
BrushOrigin is a type-safe enumeration that represents a fixed set of coordinate origins for in-game builder tools. Its primary architectural role is to enforce data integrity within the network protocol. By replacing ambiguous integer constants ("magic numbers") with a well-defined type, it prevents a class of bugs related to invalid or out-of-range values during packet serialization and deserialization.

This enum acts as a data contract between the client and server. Any packet that describes a builder tool operation will embed a BrushOrigin value to specify how the tool's shape or effect is anchored in the world. The inclusion of a static factory method, fromValue, and an instance method, getValue, establishes a clear, bidirectional serialization pattern essential for the network layer.

The use of ProtocolException for handling invalid integer values tightly couples this data type to the protocol's error handling mechanism, ensuring that malformed packets are rejected early in the processing pipeline.

### Lifecycle & Ownership
- **Creation:** Instances of BrushOrigin are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. As an enum, its instances (Center, Bottom, Top) are compile-time constants.
- **Scope:** Application-wide singleton. The instances persist for the entire lifetime of the application.
- **Destruction:** Instances are garbage collected when the application's ClassLoader is unloaded, which typically only occurs on JVM shutdown.

## Internal State & Concurrency
- **State:** Immutable. The internal integer value for each enum constant is final and assigned at compile time. The static VALUES array is also final, preventing runtime modification.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, BrushOrigin can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | static BrushOrigin | O(1) | Deserializes an integer from a network stream into a BrushOrigin instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
BrushOrigin is intended to be used as a field within a network packet definition. The protocol's serialization engine will automatically invoke getValue, while the deserialization engine will invoke fromValue.

```java
// Example of a hypothetical packet using BrushOrigin
public class PacketBuilderToolAction implements Packet {
    private BrushOrigin origin;
    // ... other fields

    @Override
    public void read(PacketBuffer buffer) {
        // The network layer reads an integer and uses fromValue to convert it
        this.origin = BrushOrigin.fromValue(buffer.readVarInt());
    }

    @Override
    public void write(PacketBuffer buffer) {
        // The network layer calls getValue to write the integer
        buffer.writeVarInt(this.origin.getValue());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Never compare BrushOrigin instances using their integer values. This defeats the purpose of type safety and makes the code brittle.

    > **Incorrect:** `if (action.getOrigin().getValue() == 0)`
    >
    > **Correct:** `if (action.getOrigin() == BrushOrigin.Center)`

- **Uncaught Exceptions:** The fromValue method can fail if it receives an undefined integer from a malformed or malicious packet. Failure to handle the resulting ProtocolException at the network boundary can lead to system instability.

- **Reflection-based Modification:** Attempting to modify the enum's internal state or instances via reflection will break the JVM's guarantees and lead to undefined behavior across the entire application.

## Data Pipeline
BrushOrigin serves as a transformation point for data moving between the game logic and the network buffer.

**Serialization (Outgoing Packet)**
> Flow:
> Game Logic State (`BrushOrigin.Center`) -> `getValue()` -> Integer (`0`) -> Network Buffer

**Deserialization (Incoming Packet)**
> Flow:
> Network Buffer -> Integer (`0`) -> `fromValue(0)` -> Game Logic State (`BrushOrigin.Center`)

