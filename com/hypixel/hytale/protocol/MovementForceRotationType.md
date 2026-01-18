---
description: Architectural reference for MovementForceRotationType
---

# MovementForceRotationType

**Package:** com.hypixel.hytale.protocol
**Type:** Enum / Utility

## Definition
```java
// Signature
public enum MovementForceRotationType {
```

## Architecture & Concepts
The MovementForceRotationType enum provides a type-safe, compile-time representation for different modes of server-enforced player rotation. It is a fundamental component of the network protocol, designed to eliminate the use of ambiguous "magic numbers" in game logic and data streams.

This enum acts as a contract between the client and server. Each constant maps to a specific integer value, which is what is actually transmitted over the network. The primary role of this class is to facilitate the serialization and deserialization of this specific movement parameter, ensuring that both endpoints interpret the data consistently. The static factory method, fromValue, is the designated entry point for converting raw network data back into a safe, usable object, while getValue is used for serialization.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. This process is automatic and occurs before any game code directly references the enum.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the JVM process. They are effectively global, immutable singletons.
- **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** Inherently immutable. The internal integer value for each constant is final and assigned at compile time. The static VALUES array is also final and populated during class initialization.
- **Thread Safety:** This class is unconditionally thread-safe. The Java Language Specification guarantees that enums are safe to access from any thread without external synchronization. Its immutable nature prevents any possibility of race conditions or state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | MovementForceRotationType | O(1) | **Static.** Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets related to player movement and camera control.

**Deserialization (Reading from a packet):**
```java
// Reading an integer value from a network buffer
int rotationTypeValue = buffer.readVarInt();

// Safely converting the integer to the enum type
try {
    MovementForceRotationType type = MovementForceRotationType.fromValue(rotationTypeValue);
    // ... apply rotation logic based on the type
} catch (ProtocolException e) {
    // Handle a corrupt or malicious packet
    // This is a critical error path
}
```

**Serialization (Writing to a packet):**
```java
// Writing the enum's integer value to a network buffer
MovementForceRotationType type = MovementForceRotationType.CameraRotation;
buffer.writeVarInt(type.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Do not use the raw integer values (0, 1, 2) directly in game logic. This defeats the purpose of the enum and creates brittle code that is difficult to maintain.
- **Ignoring Exceptions:** Failure to catch the ProtocolException from the fromValue method can lead to unhandled exceptions and client/server disconnects when receiving invalid data. Always wrap deserialization logic in a try-catch block.

## Data Pipeline
This enum serves as a data model within the network pipeline. It does not process data itself but represents a state that is serialized and deserialized.

> **Outbound Flow (Serialization):**
> Game Logic State -> **MovementForceRotationType.getValue()** -> Integer written to Network Buffer -> Sent to Network

> **Inbound Flow (Deserialization):**
> Raw Network Data -> Integer read from Network Buffer -> **MovementForceRotationType.fromValue(int)** -> Game Logic State

