---
description: Architectural reference for ApplyForceState
---

# ApplyForceState

**Package:** com.hypixel.hytale.protocol
**Type:** Data Type / Utility

## Definition
```java
// Signature
public enum ApplyForceState {
```

## Architecture & Concepts
ApplyForceState is a type-safe enumeration that represents the discrete states governing the application of physical forces to an entity. It is a fundamental component of the network protocol, designed to serialize physics-related state between the server and client efficiently.

Instead of transmitting raw integers or strings, which are prone to errors and inconsistencies, the engine uses this enum. The integer value associated with each state (e.g., Ground is 1) is used for network transport, minimizing packet size. The static factory method, fromValue, provides a robust deserialization mechanism on the receiving end, ensuring that only valid states are reconstructed.

This class prevents the use of "magic numbers" in the physics and networking code, improving readability and maintainability. If an invalid integer is received over the network, the system fails fast by throwing a ProtocolException, preventing corrupted state from propagating further into the game simulation.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants.
- **Scope:** Application-wide. As static final instances, they exist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded when the application's ClassLoader is garbage collected, typically upon application shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant is a singleton instance with a final *value* field. Its state cannot be changed after creation.
- **Thread Safety:** **Fully thread-safe**. Due to their immutable and static nature, ApplyForceState constants can be safely accessed and passed between any threads without requiring synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation used for network serialization. |
| fromValue(int value) | ApplyForceState | O(1) | **[Deserialization]** Converts a network integer back to an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used for serializing and deserializing network packets related to entity physics and for controlling logic in state machines.

```java
// Example: Deserializing a state from a network buffer
int stateValue = networkBuffer.readVarInt();
ApplyForceState currentState = ApplyForceState.fromValue(stateValue);

// Example: Using the state in a game logic switch
switch (currentState) {
    case Ground:
        // Apply friction
        break;
    case Waiting:
        // No physics update needed
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinal:** Do not use the built-in `ordinal()` method for serialization. The explicit `value` field is the canonical representation for network traffic. The `ordinal()` value can change if the declaration order of the enum constants is modified, which would break network compatibility.
- **Ignoring Exceptions:** The fromValue method can throw a ProtocolException. This is a critical error indicating data corruption or a client/server version mismatch. It must not be caught and ignored, as this will lead to undefined behavior.

## Data Pipeline
ApplyForceState serves as a bridge between the high-level game simulation and the low-level network protocol. It ensures that physics state is communicated reliably and efficiently.

> Flow:
> Server Physics State -> **ApplyForceState.Ground** -> `getValue()` -> `1` -> Network Packet -> Client Packet Decoder -> `fromValue(1)` -> **ApplyForceState.Ground** -> Client-side Entity Simulation

