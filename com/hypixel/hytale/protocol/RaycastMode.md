---
description: Architectural reference for RaycastMode
---

# RaycastMode

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum RaycastMode {
```

## Architecture & Concepts
RaycastMode is a type-safe enumeration that defines a contract for a specific gameplay mechanic within the network protocol. It translates a low-level integer, suitable for network transmission, into a high-level, self-documenting constant for use within the game engine's logic.

The primary architectural function of this enum is to eliminate "magic numbers" from the codebase. Instead of game logic checking for `raycastType == 1`, it can perform a more readable and robust check against `RaycastMode.FollowLook`. This decouples the core game logic from the specific integer values defined by the network protocol, allowing the protocol to evolve without breaking game systems.

It serves as a data model and a serialization/deserialization utility, bridging the raw data stream from the network with the strongly-typed domain of the game engine.

### Lifecycle & Ownership
- **Creation:** Enum instances are created and managed by the Java Virtual Machine (JVM) during class loading. A single, static instance of each constant, such as FollowMotion, is created and reused throughout the application's lifetime.
- **Scope:** Application-scoped. These instances persist for as long as the application is running.
- **Destruction:** Instances are destroyed only when the application's ClassLoader is garbage collected, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final integer value that is assigned at compile time and cannot be changed. The static VALUES array is populated once during static initialization and is never modified thereafter.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, RaycastMode constants can be safely accessed, passed, and read from any thread without requiring locks or other synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization between the integer protocol value and the enum type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its network protocol integer representation. |
| fromValue(int value) | RaycastMode | O(1) | Deserializes an integer from the network into a RaycastMode instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used when deserializing network packets that specify a raycasting behavior. The fromValue method is the designated factory for converting the raw network integer into a usable game object.

```java
// How a developer should normally use this
int modeFromPacket = readIntFromBuffer();
try {
    RaycastMode mode = RaycastMode.fromValue(modeFromPacket);
    // ... apply game logic based on the mode
} catch (ProtocolException e) {
    // Handle malformed packet or disconnect client
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The explicit `value` field is used to guarantee that the network protocol contract remains stable even if the declaration order of the enum constants changes.
- **Passing Integers:** Avoid passing the raw integer value through different layers of the application. Convert to the RaycastMode type at the protocol boundary and use the enum type exclusively in all subsequent game logic.

## Data Pipeline
RaycastMode acts as a deserialization gateway, converting raw network data into a structured, type-safe object for consumption by the game engine.

> Flow:
> Network Packet (Integer) -> Protocol Buffer Reader -> **RaycastMode.fromValue()** -> Game Logic System -> Entity Behavior Update

