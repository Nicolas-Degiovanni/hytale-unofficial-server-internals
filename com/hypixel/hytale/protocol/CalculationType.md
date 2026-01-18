---
description: Architectural reference for CalculationType
---

# CalculationType

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum CalculationType {
   Additive(0),
   Multiplicative(1);
```

## Architecture & Concepts
CalculationType is a foundational enumeration within the network protocol layer. It provides a type-safe, human-readable representation for how a numerical modification should be applied, typically within the context of game mechanics like attribute calculations, buffs, or item statistics.

Its primary architectural function is to decouple the game logic from the raw integer values transmitted over the network. By mapping a specific integer (e.g., 0) to a named constant (e.g., Additive), it prevents the proliferation of "magic numbers" throughout the codebase. This ensures that the network data contract is explicit, robust, and less prone to errors if the protocol evolves.

This enum is a critical component for both serialization (converting game state to a byte stream) and deserialization (reconstructing game state from a byte stream).

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during the class-loading phase. The static `VALUES` array is also initialized at this time. These instances are managed entirely by the JVM.
- **Scope:** Application-wide. The `Additive` and `Multiplicative` instances are singletons that persist for the entire lifetime of the application.
- **Destruction:** The instances are garbage collected by the JVM only when the defining ClassLoader is unloaded, which typically occurs at application shutdown. Manual destruction is not possible or necessary.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a private final `value` field that is set at creation and can never be modified. The static `VALUES` array is also final.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, CalculationType can be safely accessed, passed, and read from any number of concurrent threads without requiring any synchronization mechanisms. The static factory method `fromValue` is also fully thread-safe.

## API Surface
The public contract is minimal, focusing exclusively on conversion between the enum constant and its underlying protocol value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | CalculationType | O(1) | **[Factory]** Deserializes an integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets that contain attribute modifiers or similar game data.

**Deserialization (Reading from a packet):**
```java
// How a developer should normally use this
int rawType = packet.readVarInt();
CalculationType type = CalculationType.fromValue(rawType);

// Now use the type-safe enum in game logic
player.getAttributes().applyModifier(..., type);
```

**Serialization (Writing to a packet):**
```java
// Get the integer value for the network stream
CalculationType type = modifier.getCalculationType();
packet.writeVarInt(type.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinal Values:** Never use the built-in `ordinal()` method for serialization. The ordinal value is dependent on the declaration order and will break the network protocol if a new constant is added or the existing ones are reordered. Always use `getValue()`.
- **Ignoring Exceptions:** The `fromValue` method is a critical validation point. Failure to catch the `ProtocolException` it can throw will result in unhandled exceptions that may disconnect a client or crash a server process. Always wrap calls in a try-catch block during packet processing.

## Data Pipeline
CalculationType acts as a translation and validation step during the deserialization of game state data from the network.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> **CalculationType.fromValue(intValue)** -> Game Logic (e.g., AttributeSystem)

