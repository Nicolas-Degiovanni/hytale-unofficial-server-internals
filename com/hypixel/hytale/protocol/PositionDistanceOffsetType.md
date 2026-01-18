---
description: Architectural reference for PositionDistanceOffsetType
---

# PositionDistanceOffsetType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum PositionDistanceOffsetType {
```

## Architecture & Concepts
PositionDistanceOffsetType is a type-safe enumeration that defines the method for calculating a positional offset within the Hytale network protocol. It serves as a contract between the client and server, ensuring that both systems interpret positional data in a consistent manner.

This enum is a fundamental building block of the protocol layer, used to disambiguate fields within larger packets, likely related to entity interaction, block placement, or ability targeting. By encoding the calculation *method* as a simple integer, the protocol remains efficient while allowing for complex and varied game logic on either end.

The defined types are:
*   **DistanceOffset:** Represents a simple, direct vector offset from a source position.
*   **DistanceOffsetRaycast:** Represents a more computationally intensive offset determined by a server-side raycast. This is typically used for targeting surfaces or entities in the world.
*   **None:** Indicates that no positional offset should be applied, and the base position should be used as-is.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during class loading. They are compile-time constants and exist for the entire application lifecycle. The static factory method fromValue does not create new instances but returns one of the pre-existing singletons.
- **Scope:** Application-wide. These constants are static and globally accessible.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual memory management required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a private, final integer value that is set at compile time. Its state cannot be modified at runtime.
- **Thread Safety:** **Inherently Thread-Safe**. Due to their immutable nature and the JVM's guarantees for enum initialization, these constants can be safely accessed from any thread without external synchronization.

## API Surface
The public contract is designed for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | PositionDistanceOffsetType | O(1) | Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet serialization and deserialization to interpret a data field correctly.

```java
// Example: Deserializing a packet field
int rawOffsetType = networkBuffer.readVarInt();
PositionDistanceOffsetType offsetType = PositionDistanceOffsetType.fromValue(rawOffsetType);

switch (offsetType) {
    case DistanceOffset:
        // handle simple vector offset
        break;
    case DistanceOffsetRaycast:
        // handle raycast-based offset
        break;
    case None:
        // apply no offset
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never use the raw integer values (0, 1, 2) directly in game logic. This defeats the purpose of type safety and creates brittle code that is difficult to refactor. Always reference the enum constants directly, for example, `PositionDistanceOffsetType.DistanceOffset`.
- **Ignoring Deserialization Errors:** The fromValue method throws a checked ProtocolException for invalid integer inputs. This exception **must** be caught and handled. Ignoring it can lead to a disconnect or crash the network processing thread, as it indicates a corrupted or malicious packet.

## Data Pipeline
PositionDistanceOffsetType acts as a data contract during network transmission. It is not a processing stage itself but rather defines how subsequent stages should interpret positional data.

**Serialization Flow (Client -> Server)**
> Game Logic -> **PositionDistanceOffsetType.DistanceOffsetRaycast** -> getValue() -> `1` -> Network Buffer

**Deserialization Flow (Server <- Client)**
> Network Buffer -> `1` -> fromValue(1) -> **PositionDistanceOffsetType.DistanceOffsetRaycast** -> Game Logic (Raycast System)

