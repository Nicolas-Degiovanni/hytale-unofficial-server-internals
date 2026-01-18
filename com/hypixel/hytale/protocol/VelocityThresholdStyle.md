---
description: Architectural reference for VelocityThresholdStyle
---

# VelocityThresholdStyle

**Package:** com.hypixel.hytale.protocol
**Type:** Enum / Utility

## Definition
```java
// Signature
public enum VelocityThresholdStyle {
```

## Architecture & Concepts
VelocityThresholdStyle is a type-safe enumeration that defines a constrained set of algorithms for calculating velocity thresholds within the physics or gameplay engine. Its primary architectural role is to serve as a data contract for network communication, ensuring that client and server agree on a specific, well-defined behavior.

By mapping descriptive names (Linear, Exp) to compact integer representations, this enum eliminates the use of ambiguous "magic numbers" in the protocol stream. It acts as a serialization and deserialization utility, converting between the network-efficient integer and the type-safe object used by the game logic. The integration with ProtocolException makes it a critical component of the network layer's data validation and error-handling strategy.

## Lifecycle & Ownership
- **Creation:** The enum constants, Linear and Exp, are instantiated automatically by the Java Virtual Machine during class loading. This process is guaranteed to happen only once. The static VALUES array is also initialized at this time.
- **Scope:** Application-wide. The enum instances are static final constants that persist for the entire lifetime of the application.
- **Destruction:** The instances are managed by the JVM and are unloaded along with their class loader during application shutdown. Manual destruction is not possible or necessary.

## Internal State & Concurrency
- **State:** Deeply immutable. The internal state of each enum constant, including its name and integer value, is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of VelocityThresholdStyle can be safely accessed, shared, and passed between any number of threads without requiring external synchronization or locks. All static methods are re-entrant and operate on immutable data, guaranteeing their safety in concurrent environments.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | VelocityThresholdStyle | O(1) | Deserializes an integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a value received from a network packet. The fromValue method is the designated factory for this purpose.

```java
// How a developer should deserialize this value
int receivedValue = networkBuffer.readVarInt();
try {
    VelocityThresholdStyle style = VelocityThresholdStyle.fromValue(receivedValue);
    // ... apply gameplay logic using the 'style' object
} catch (ProtocolException e) {
    // Handle corrupt or invalid data from the client/server
    connection.disconnect("Invalid VelocityThresholdStyle");
}
```

### Anti-Patterns (Do NOT do this)
- **Unsafe Deserialization:** Do not use the static VALUES array directly with an index from the network. This bypasses the critical bounds-checking logic in fromValue and can lead to an ArrayIndexOutOfBoundsException, causing an unhandled crash.
- **Ignoring Exceptions:** Failure to catch the ProtocolException thrown by fromValue will result in an unhandled exception that can terminate the network processing thread. Always wrap calls to fromValue in a try-catch block when processing external data.

## Data Pipeline
VelocityThresholdStyle serves as a translation point between the high-level game state and the low-level network stream.

> Flow:
> Server Game Logic -> **VelocityThresholdStyle** instance -> `getValue()` -> Integer written to Network Buffer -> Client Packet Decoder -> `fromValue(int)` -> **VelocityThresholdStyle** instance -> Client Game Logic

