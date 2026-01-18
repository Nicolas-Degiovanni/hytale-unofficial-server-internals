---
description: Architectural reference for PositionType
---

# PositionType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum PositionType {
```

## Architecture & Concepts
PositionType is a protocol-level enumeration that defines the strategy for interpreting positional data within network packets. It serves as a type-safe discriminator, ensuring that both the client and server agree on how to calculate an entity's or object's location in the world.

This enumeration is fundamental to the network layer's data serialization and deserialization process. By encoding a position calculation strategy as a simple integer, it minimizes packet size while maintaining protocol clarity and preventing the use of ambiguous "magic numbers".

The two defined strategies are:
*   **AttachedToPlusOffset:** Indicates the position is relative to another entity, plus a specific offset. This is highly efficient for child entities or objects attached to a parent.
*   **Custom:** Indicates the position is an absolute, standalone coordinate within the world space.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. They are created once and exist as static singletons.
- **Scope:** Application-wide. The PositionType constants persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are garbage collected by the JVM only when the application's class loader is unloaded, typically during a full shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The internal integer value for each enum constant is final and cannot be changed after instantiation. The static VALUES array is also final.
- **Thread Safety:** PositionType is unconditionally thread-safe. As an immutable, JVM-managed construct, it can be safely accessed and used from any thread without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | PositionType | O(1) | **Critical.** Deserializes an integer from a network stream into a PositionType instance. Throws ProtocolException on an invalid or out-of-bounds value. |

## Integration Patterns

### Standard Usage
This enum is primarily used by packet serialization and deserialization logic. The `fromValue` method is the designated entry point for converting raw network data into a type-safe object.

**Deserialization (Reading a packet):**
```java
// In a packet decoder, after reading an integer from the buffer
int rawType = buffer.readVarInt();
PositionType type = PositionType.fromValue(rawType);

// Now, the logic can safely switch on the type
switch (type) {
    case AttachedToPlusOffset:
        // read parent entity ID and offset vector
        break;
    case Custom:
        // read absolute world coordinates
        break;
}
```

**Serialization (Writing a packet):**
```java
// In a packet encoder
PositionType type = ...; // The type to be written
buffer.writeVarInt(type.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Manual Value Checking:** Do not use `if (type.getValue() == 0)`. This defeats the purpose of a type-safe enum. Always use `==` or a switch statement on the enum constants themselves (e.g., `if (type == PositionType.AttachedToPlusOffset)`).
- **Ignoring Exceptions:** The `fromValue` method is a critical validation point. Failure to catch its ProtocolException can lead to unhandled exceptions in the network thread, potentially disconnecting the client due to a malformed or malicious packet.

## Data Pipeline
PositionType acts as a translator between the raw binary protocol and the engine's internal game logic.

> **Incoming Flow (Deserialization):**
> Raw Byte Stream -> Protocol Decoder reads integer -> **PositionType.fromValue(integer)** -> PositionType Instance -> Game State Update

> **Outgoing Flow (Serialization):**
> Game State Logic -> PositionType Instance -> **instance.getValue()** -> Integer written by Protocol Encoder -> Raw Byte Stream

