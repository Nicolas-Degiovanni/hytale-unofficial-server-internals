---
description: Architectural reference for RotationMode
---

# RotationMode

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum RotationMode {
```

## Architecture & Concepts
The RotationMode enum defines a constrained, type-safe set of behaviors for entity rotation within the game engine. It serves as a critical component of the network protocol, translating a low-level integer value from the data stream into a high-level, self-documenting game state.

Its primary architectural function is to act as a serialization and deserialization bridge. By mapping specific rotation algorithms (e.g., direct velocity-based, damped) to fixed integer identifiers, it eliminates the use of "magic numbers" in the networking and game logic layers. This ensures that both the client and server have a shared, unambiguous understanding of how an entity's orientation should be calculated and rendered.

The static factory method, fromValue, is the designated entry point for data arriving from the network, providing immediate validation and conversion into a usable game object.

## Lifecycle & Ownership
-   **Creation:** Instances are created by the Java Virtual Machine (JVM) during class loading. They are compile-time constants and cannot be instantiated at runtime.
-   **Scope:** Application-scoped. The enum constants (None, Velocity, VelocityDamped, VelocityRoll) exist for the entire lifetime of the application.
-   **Destruction:** Cleaned up by the JVM when the class loader is garbage collected, which typically occurs only at application shutdown.

## Internal State & Concurrency
-   **State:** Immutable. Each enum constant holds a private final integer, which is set at compile time and cannot be modified. The static VALUES array is also final and serves as a read-only cache for deserialization.
-   **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of RotationMode can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant, used for network serialization. |
| fromValue(int value) | static RotationMode | O(1) | Deserializes an integer from a network stream into a RotationMode instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case involves deserializing an integer from a network packet and then using the resulting enum in a control flow statement to apply the correct game logic.

```java
// Example: Processing an entity update packet
int rotationModeId = packet.readVarInt();
RotationMode mode = RotationMode.fromValue(rotationModeId);

// Apply game logic based on the deserialized mode
switch (mode) {
    case Velocity:
        entity.setRotationFromVelocity();
        break;
    case VelocityDamped:
        entity.setRotationFromDampedVelocity();
        break;
    case None:
        // No rotational update is applied
        break;
    default:
        // Handle other cases
        break;
}
```

### Anti-Patterns (Do NOT do this)
-   **Magic Numbers:** Avoid using the raw integer values (0, 1, 2, 3) directly in game logic. This defeats the purpose of the enum, creating brittle code that is difficult to read and maintain. Always compare against the enum constants (e.g., `if (mode == RotationMode.Velocity)`).
-   **Reflection:** Do not attempt to create new instances of this enum via reflection. This violates the fundamental contract of enums in Java and will lead to unpredictable behavior and potential JVM instability.

## Data Pipeline
RotationMode is a data-level component that exists at the boundary between the raw network stream and the game state simulation.

> Flow:
> Network Byte Stream -> Protocol Decoder (reads integer) -> **RotationMode.fromValue(intValue)** -> Entity State Object -> Entity Update System

