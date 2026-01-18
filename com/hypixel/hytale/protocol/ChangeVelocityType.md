---
description: Architectural reference for ChangeVelocityType
---

# ChangeVelocityType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum ChangeVelocityType {
```

## Architecture & Concepts
ChangeVelocityType is a type-safe enumeration that represents how an entity's velocity should be modified. It is a core component of the network protocol layer, responsible for translating low-level integer identifiers from the byte stream into high-level, readable, and compile-time-checked constants.

This enum serves as a data contract between the client and server. Its primary function is to eliminate "magic numbers" within the physics and entity update systems, ensuring that any code handling velocity changes can only operate on a well-defined set of behaviors: adding to the current velocity or setting it to a new value. The inclusion of the static factory method, fromValue, centralizes the deserialization logic and enforces strict validation against invalid or unsupported protocol data.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. The static VALUES array is also populated at this time. These instances are not created by application code.
- **Scope:** Application-wide. As static final constants, Add and Set persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded from memory only when the JVM shuts down. There is no manual destruction or garbage collection concern.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final primitive integer. The static VALUES array is populated once and is not modified thereafter. The class itself is stateless.
- **Thread Safety:** This class is inherently thread-safe. All state is final and established during static initialization. The fromValue method is a pure function operating on immutable data and is safe for concurrent invocation from any thread without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for serialization. |
| fromValue(int value) | static ChangeVelocityType | O(1) | Deserializes an integer into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is typically used during packet serialization and deserialization. The game logic then uses the enum constants for clear, readable conditional branching.

```java
// Deserialization from a network packet
int rawType = buffer.readVarInt();
ChangeVelocityType type = ChangeVelocityType.fromValue(rawType);

// Game logic based on the type
if (type == ChangeVelocityType.Set) {
    entity.setVelocity(newVelocity);
} else {
    entity.addVelocity(newVelocity);
}

// Serialization to a network packet
buffer.writeVarInt(ChangeVelocityType.Set.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Value:** Do not compare enum instances using their underlying integer value. This defeats the purpose of type safety and introduces magic numbers back into the code.

```java
// AVOID THIS
if (type.getValue() == 0) {
    // ... logic for Add
}

// PREFER THIS
if (type == ChangeVelocityType.Add) {
    // ... logic for Add
}
```
- **Ignoring Protocol Exceptions:** The fromValue method validates incoming data. Swallowing its exception can lead to silent data corruption or undefined behavior in the physics engine. The calling code in the protocol layer is responsible for handling a malformed packet.

## Data Pipeline
ChangeVelocityType acts as a deserialization and validation gate for a specific field within a larger data flow.

> Flow:
> Network Byte Stream -> Integer Field Deserialization -> **ChangeVelocityType.fromValue()** -> Packet Object Field -> Entity System Logic

