---
description: Architectural reference for RotationType
---

# RotationType

**Package:** com.hypixel.hytale.protocol
**Type:** Enum (Static Data Type)

## Definition
```java
// Signature
public enum RotationType {
```

## Architecture & Concepts
RotationType is a protocol-level enumeration that defines how an entity's rotation is calculated and transmitted over the network. It serves as a type-safe, fixed-size representation for different rotation behaviors, abstracting the underlying integer value used for serialization.

This enum is a critical component of the entity state synchronization system. By mapping a human-readable constant like **AttachedToPlusOffset** to a compact integer (0), it ensures that network packets remain small and efficient. The static factory method, fromValue, is the primary entry point for deserialization, converting a raw integer from an incoming data stream back into a strongly-typed object. This prevents invalid state from propagating through the game engine by immediately throwing a ProtocolException if an undefined value is received.

## Lifecycle & Ownership
- **Creation:** All enum constants (AttachedToPlusOffset, Custom) are instantiated by the Java Virtual Machine during the initial loading of the RotationType class. This process is managed entirely by the JVM and occurs only once.
- **Scope:** Application-wide. The enum constants are static, final singletons that persist for the entire lifetime of the application.
- **Destruction:** The constants are eligible for garbage collection only when the RotationType class is unloaded, which typically occurs only at application shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. Each enum constant is a singleton instance with a final internal state (the integer value) that cannot be modified after creation.
- **Thread Safety:** Fully thread-safe. As immutable singletons, RotationType constants can be safely accessed, passed, and read from any thread without requiring external synchronization or locks. The static methods are also stateless and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for network serialization. |
| fromValue(int value) | static RotationType | O(1) | Deserializes an integer into a RotationType constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of entity data.

**Serialization (Writing to a network buffer):**
```java
// Get the integer value to send over the network
RotationType type = RotationType.AttachedToPlusOffset;
int valueToSend = type.getValue();
networkBuffer.writeInt(valueToSend);
```

**Deserialization (Reading from a network buffer):**
```java
// Read the integer and convert it back to a type-safe enum
int receivedValue = networkBuffer.readInt();
RotationType type = RotationType.fromValue(receivedValue);

// Use the type in game logic
if (type == RotationType.Custom) {
    // handle custom rotation logic
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinal for Serialization:** Do not use the built-in `ordinal()` method for serialization. The integer value returned by `ordinal()` is dependent on the declaration order of the constants. If a new constant is added in the middle, all subsequent ordinals will shift, breaking network compatibility. Always use the explicit `getValue()` method.
- **Ignoring Exceptions:** The `fromValue` method can throw a ProtocolException. This is a critical failure indicating a corrupt or mismatched protocol version. This exception must be caught at the protocol layer to handle the invalid data, typically by disconnecting the client.

## Data Pipeline
RotationType is not a processing component but rather a data model that flows through the network and game state pipelines.

> Flow:
> Server Entity State -> Serializer (uses **getValue()**) -> Network Packet (int) -> Client Deserializer (uses **fromValue()**) -> **RotationType (enum)** -> Entity Component State -> Rendering System

