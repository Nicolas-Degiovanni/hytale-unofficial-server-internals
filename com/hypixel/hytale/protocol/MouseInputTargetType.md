---
description: Architectural reference for MouseInputTargetType
---

# MouseInputTargetType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum MouseInputTargetType
```

## Architecture & Concepts
MouseInputTargetType is a type-safe enumeration that serves as a serialization contract within the Hytale network protocol. Its primary function is to translate a low-level integer, received from a network packet, into a high-level, self-describing game concept representing what a player's mouse is interacting with.

This class is fundamental to the input processing pipeline. It eliminates the use of "magic numbers" (e.g., using the integer 1 to represent a block) in the game logic, replacing them with readable constants like MouseInputTargetType.Block. This improves code clarity and reduces the risk of errors.

The static factory method, fromValue, acts as a deserialization gatekeeper. It validates incoming data, ensuring that only defined integer values are processed further. Any invalid value results in a ProtocolException, allowing the network layer to reject malformed packets early and maintain protocol integrity.

### Lifecycle & Ownership
- **Creation:** Enum instances are created and initialized by the Java Virtual Machine during class loading, before any application code executes. The static VALUES array is also populated at this time as a one-time performance optimization.
- **Scope:** Application-scoped. The enum constants (Any, Block, Entity, None) are effectively static singletons that persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded when the application's class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final integer value that is assigned at creation and can never be modified. The static VALUES array is populated once and is never written to again.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability, instances can be safely shared and read across any number of threads without requiring locks or other synchronization mechanisms.

## API Surface
The public contract is focused on serialization and deserialization between the integer representation and the enum type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromValue(int value) | MouseInputTargetType | O(1) | **Static Factory.** Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |
| getValue() | int | O(1) | Serializes the enum constant into its integer representation for network transmission. |

## Integration Patterns

### Standard Usage
This enum is typically used within protocol handlers or deserializers to interpret data read from a network buffer.

```java
// Example: Deserializing an input packet
int rawTargetType = buffer.readVarInt();
MouseInputTargetType target = MouseInputTargetType.fromValue(rawTargetType);

// Game logic now uses the type-safe enum
if (target == MouseInputTargetType.Entity) {
    // process entity interaction...
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Do not use the integer values directly in game logic. Comparing against `target.getValue() == 1` defeats the purpose of the enum and re-introduces magic numbers.
- **Ignoring Exceptions:** Failure to handle the ProtocolException thrown by fromValue can lead to unhandled exceptions in the network processing thread, potentially causing a client disconnection. Always wrap calls in a try-catch block when processing untrusted network data.

## Data Pipeline
MouseInputTargetType is a critical component in the input data deserialization pipeline, converting raw network data into a structured, usable format for the game engine.

> Flow:
> Network Packet (contains integer) -> Protocol Buffer Reader -> **MouseInputTargetType.fromValue()** -> Game Input Event -> Game Logic System

