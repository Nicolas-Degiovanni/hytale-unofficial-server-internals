---
description: Architectural reference for CurveType
---

# CurveType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum CurveType {
```

## Architecture & Concepts
CurveType is a type-safe enumeration that represents different animation or interpolation curves used within the Hytale protocol. Its primary architectural function is to act as a strict contract between the raw integer values sent over the network and their meaningful, self-documenting representations within the game engine.

This enum is a critical component of the protocol's serialization and deserialization layer. It prevents the use of "magic numbers" in the application logic, replacing ambiguous integers like 0 or 1 with explicit, readable constants like CurveType.Linear or CurveType.QuartIn.

The static factory method, fromValue, serves as the designated deserialization entry point. It includes essential boundary checks, throwing a ProtocolException for any undefined integer value. This ensures that corrupted or malformed network packets are rejected early, preventing invalid state from propagating deeper into the game systems.

### Lifecycle & Ownership
- **Creation:** All enum instances (Linear, QuartIn, etc.) are constructed and initialized by the JVM during class loading. This process is automatic and occurs once per application lifecycle.
- **Scope:** Application-wide. The instances are static and persist for the entire duration of the JVM process.
- **Destruction:** The enum instances are garbage collected when the class loader is unloaded, which typically only happens upon JVM shutdown. There is no manual memory management.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant has a final `value` field that is set at creation time and can never be changed. The static VALUES array is also populated once and is never modified thereafter.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, CurveType can be safely accessed, passed, and read from any thread without requiring locks or other synchronization primitives. All methods are re-entrant and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the protocol-defined integer for this curve type. Used for serialization. |
| fromValue(int value) | CurveType | O(1) | **Deserialization Gateway.** Converts a raw integer from a network stream into a CurveType instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
CurveType is used whenever a system needs to serialize or deserialize an animation curve identifier. The primary interaction points are the getValue and fromValue methods.

```java
// Deserialization: Reading from a network buffer
int curveId = buffer.readVarInt();
CurveType curve = CurveType.fromValue(curveId);

// Game Logic: Using the type-safe enum
switch (curve) {
    case Linear:
        // handle linear interpolation
        break;
    case QuartIn:
        // handle QuartIn interpolation
        break;
    default:
        // ...
}

// Serialization: Writing to a network buffer
CurveType curveToUse = CurveType.QuartInOut;
buffer.writeVarInt(curveToUse.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using `ordinal()` for Serialization:** Never use the built-in `ordinal()` method to get the integer value for serialization. The ordinal value is based on declaration order and is extremely brittle; reordering the constants in the source file would break network compatibility. Always use the explicit `getValue()` method.
- **Ignoring ProtocolException:** The `fromValue` method is a critical validation step. Failure to catch the ProtocolException it can throw will result in an unhandled exception that may terminate the network connection or crash the client/server thread.
- **Manual Value Checking:** Do not write code that manually checks if an integer is valid before calling `fromValue`. The method is designed to perform this validation itself. Rely on its contract.

## Data Pipeline
CurveType acts as a transformation and validation step during data deserialization from the network.

> Flow:
> Network Byte Stream -> Protocol Decoder (reads integer) -> **CurveType.fromValue(intValue)** -> Game Logic (Animation System)

