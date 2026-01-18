---
description: Architectural reference for AttachedToType
---

# AttachedToType

**Package:** com.hypixel.hytale.protocol
**Type:** Type Definition

## Definition
```java
// Signature
public enum AttachedToType {
```

## Architecture & Concepts
AttachedToType is a foundational enumeration within the Hytale network protocol layer. Its primary function is to provide a type-safe, compile-time constant representation for an integer value that signifies how an object, often the camera or a UI element, is attached within the game world.

This enum serves as a critical serialization and deserialization contract. It translates a raw integer from a network packet into a well-defined, readable state that can be safely used by higher-level game systems. By abstracting away "magic numbers" (0, 1, 2), it prevents a common class of bugs related to network protocol mismatches and improves code clarity. It is a core component for interpreting entity state and player perspective packets.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. There is no manual instantiation; they are compile-time constructs.
- **Scope:** Static. The enum and its constants exist for the entire lifetime of the application once the AttachedToType class has been loaded by the ClassLoader.
- **Destruction:** The constants are eligible for garbage collection only when the defining ClassLoader is itself collected, which typically occurs only at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant is a singleton instance with a final internal state (the integer value). This state cannot be modified after creation.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of AttachedToType can be passed between and accessed by multiple threads without any need for external synchronization or locks. The static factory method fromValue is also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LocalPlayer | AttachedToType | O(1) | Constant representing an attachment to the local player's context. |
| EntityId | AttachedToType | O(1) | Constant representing an attachment to a specific world entity. |
| None | AttachedToType | O(1) | Constant representing a detached or default state. |
| getValue() | int | O(1) | Serializes the enum constant to its integer representation for network transmission. |
| fromValue(int) | static AttachedToType | O(1) | Deserializes an integer from a network stream into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used when reading from or writing to a network buffer. The fromValue method is the designated entry point for deserialization, and getValue is used for serialization.

```java
// Deserializing an attachment type from a network buffer
int rawType = buffer.readVarInt();
AttachedToType attachment = AttachedToType.fromValue(rawType);

// Using the type in game logic
switch (attachment) {
    case LocalPlayer:
        camera.attachTo(player);
        break;
    case EntityId:
        int entityId = buffer.readVarInt();
        camera.attachTo(world.getEntity(entityId));
        break;
    case None:
        camera.detach();
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Numeric Comparison:** Do not compare enum instances using their integer values. This defeats the purpose of type safety and re-introduces magic numbers.
    - **BAD:** `if (attachment.getValue() == 0)`
    - **GOOD:** `if (attachment == AttachedToType.LocalPlayer)`
- **Unsafe Deserialization:** Do not bypass the fromValue method. The built-in bounds checking is critical for protocol stability and security. A malicious or malformed packet could otherwise cause an ArrayIndexOutOfBoundsException.
- **Ignoring Exceptions:** The ProtocolException thrown by fromValue is a non-recoverable error indicating a corrupt or mismatched protocol stream. It must be caught at the network layer and should result in the client's disconnection.

## Data Pipeline
The flow for this type is unidirectional during packet processing: from a raw network value to a structured, in-engine type.

> Flow:
> Network Byte Stream -> Protocol Buffer Decoder -> Raw Integer -> **AttachedToType.fromValue()** -> Game Logic (e.g., CameraSystem, PlayerController)

