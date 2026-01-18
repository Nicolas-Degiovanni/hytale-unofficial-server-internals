---
description: Architectural reference for ConnectedBlockRuleSetType
---

# ConnectedBlockRuleSetType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum ConnectedBlockRuleSetType {
```

## Architecture & Concepts
The ConnectedBlockRuleSetType is a type-safe enumeration that defines the specific rendering rules for visually connected blocks, such as stairs and roofs. Its primary function is to act as a serialization and deserialization contract within the network protocol layer.

This enum translates a compact integer identifier, received from the network stream, into a strongly-typed, self-documenting object for use within the client's rendering and block logic systems. By replacing ambiguous "magic numbers" (e.g., 0 for Stair) with explicit constants (e.g., ConnectedBlockRuleSetType.Stair), it enhances code readability, reduces the risk of logic errors, and centralizes the protocol definition.

The inclusion of a static `fromValue` factory method with strict bounds checking makes this enum a critical component for protocol validation, ensuring that malformed or malicious data packets are rejected early in the processing pipeline.

### Lifecycle & Ownership
- **Creation:** All enum constants (Stair, Roof) are instantiated automatically by the Java Virtual Machine (JVM) during the initial class loading phase. This occurs once, very early in the application lifecycle.
- **Scope:** Application-scoped. The instances are static and persist for the entire lifetime of the application. They are effectively permanent, globally accessible singletons.
- **Destruction:** The enum instances are never explicitly destroyed. They are garbage collected only when the JVM itself shuts down.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a private final `value` field that is set at compile time and cannot be modified. The static `VALUES` array, a cache for performance, is also effectively immutable after its one-time initialization.
- **Thread Safety:** This class is inherently thread-safe. As an immutable object, it can be safely read, passed, and referenced by multiple threads simultaneously without any need for external synchronization or locking mechanisms.

## API Surface
The public contract is minimal, focused exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | ConnectedBlockRuleSetType | O(1) | **Static Factory.** Deserializes an integer into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used when reading from or writing to a network buffer that contains block data.

**Deserialization (Reading from Network):**
```java
// A value is read from an incoming network buffer
int ruleSetTypeId = networkBuffer.readVarInt();

// The integer is converted to a safe type for game logic
try {
    ConnectedBlockRuleSetType ruleSet = ConnectedBlockRuleSetType.fromValue(ruleSetTypeId);
    // ... use ruleSet in block rendering logic
} catch (ProtocolException e) {
    // Handle corrupt or malicious packet data
    connection.disconnect("Invalid block data received.");
}
```

**Serialization (Writing to Network):**
```java
// An object in the game world has a specific rule set
ConnectedBlockRuleSetType ruleSet = myBlock.getConnectedRuleSet();

// The type is converted to its integer value for transmission
networkBuffer.writeVarInt(ruleSet.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Never rely on the `ordinal()` method. The integer value for the protocol is explicitly defined and retrieved via `getValue()`. The ordinal is an internal implementation detail and can change, breaking network compatibility.
- **Ignoring Exceptions:** Failure to catch the ProtocolException thrown by `fromValue` can lead to an unhandled exception that crashes the client session when processing a malformed packet. Always wrap calls to `fromValue` in a try-catch block.
- **Using Magic Numbers:** Avoid using raw integers like `0` or `1` in game logic. This defeats the purpose of the enum and creates brittle code. Always compare against the enum constants directly (e.g., `if (ruleSet == ConnectedBlockRuleSetType.Stair)`).

## Data Pipeline
ConnectedBlockRuleSetType serves as a translation point between the raw network layer and the typed game logic layer.

> **Inbound Flow (Deserialization):**
> Network Byte Stream → Protocol Deserializer → `fromValue(intValue)` → **ConnectedBlockRuleSetType Instance** → Block Entity State → Rendering Engine

> **Outbound Flow (Serialization):**
> Game State Change → Block Entity State → **ConnectedBlockRuleSetType Instance** → `getValue()` → Protocol Serializer → Network Byte Stream

