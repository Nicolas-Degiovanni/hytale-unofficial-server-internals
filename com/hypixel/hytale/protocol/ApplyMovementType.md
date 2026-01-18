---
description: Architectural reference for ApplyMovementType
---

# ApplyMovementType

**Package:** com.hypixel.hytale.protocol
**Type:** Type Enum

## Definition
```java
// Signature
public enum ApplyMovementType {
```

## Architecture & Concepts
ApplyMovementType is a type-safe enumeration that defines the strategy for applying a client's movement update on the server. It is a fundamental component of the entity movement protocol, ensuring that both client and server agree on how position data is interpreted.

This enum prevents the use of ambiguous "magic numbers" in the network stream. Instead of sending raw integers like 0 or 1, the protocol logic serializes and deserializes through this enum, providing compile-time safety and clear intent.

- **CharacterController (0):** Instructs the receiving system to process the movement data through the standard physics and collision simulation. This is the default mode for continuous player input.
- **Position (1):** Instructs the receiving system to bypass the character controller and directly set the entity's position. This is typically used for teleportation events or administrative commands.

Its location in the `com.hypixel.hytale.protocol` package signifies its exclusive role in network serialization and state synchronization.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. Individual instances are not created by application code.
- **Scope:** Application-wide static constants. They persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are garbage collected when the application's ClassLoader is unloaded, typically during shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant is a singleton with a final `value` field. The static `VALUES` array is also final and populated only once at class initialization.
- **Thread Safety:** Inherently thread-safe. As an immutable, statically initialized type, ApplyMovementType can be safely accessed and used by any thread without external synchronization.

## API Surface
The public contract is designed for serialization and deserialization within the network protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | ApplyMovementType | O(1) | Deserializes an integer from the network into an enum constant. Throws ProtocolException for invalid values. |

## Integration Patterns

### Standard Usage
This enum is used when serializing or deserializing network packets that contain movement information. The `fromValue` method is the designated factory for converting raw network data back into a type-safe object.

```java
// Deserializing a movement packet from a network buffer
int rawMovementType = buffer.readVarInt();
ApplyMovementType movementType = ApplyMovementType.fromValue(rawMovementType);

// Switch logic based on the deserialized type
switch (movementType) {
    case CharacterController:
        // process standard movement
        break;
    case Position:
        // apply direct position set
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Do not use hardcoded integers (0, 1) in game logic. This creates brittle code and defeats the purpose of the enum. Always reference the constants directly, for example, `ApplyMovementType.CharacterController`.
- **Ignoring Exceptions:** The `fromValue` method throws a ProtocolException on invalid input. This is a critical error indicating data corruption or a protocol version mismatch. It must not be ignored. A client receiving such an error should be disconnected.
- **Manual Ordinal Mapping:** Do not rely on the `ordinal()` method for serialization. The explicitly defined `value` field is the canonical representation for the network protocol and is resilient to changes in declaration order.

## Data Pipeline
ApplyMovementType is not a processing stage but rather a data field within a larger network packet structure. It dictates the processing path for movement data.

> **Outbound Flow (Serialization):**
> Game Logic -> MovementPacket(type: **ApplyMovementType.CharacterController**) -> Serializer calls `getValue()` -> Integer (0) -> Network Buffer

> **Inbound Flow (Deserialization):**
> Network Buffer -> Integer (0) -> Deserializer calls `fromValue(0)` -> **ApplyMovementType.CharacterController** -> MovementPacket -> Server Game Logic

