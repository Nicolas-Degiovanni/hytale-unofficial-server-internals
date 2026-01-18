---
description: Architectural reference for GameMode
---

# GameMode

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum GameMode {
```

## Architecture & Concepts
The GameMode enum provides a compile-time, type-safe representation for the core gameplay modes within the Hytale engine. It is a fundamental component of the network protocol, designed to eliminate "magic number" vulnerabilities and ensure that game state is transmitted and interpreted unambiguously between the client and server.

Its primary architectural role is to act as a data contract. By mapping symbolic names like **Adventure** and **Creative** to fixed integer values, it provides a stable and efficient serialization format for network packets and save files. This design prevents logic errors that could arise from using raw integers, where a value of 2, for example, would be meaningless. The static factory method, fromValue, serves as a strict deserialization gateway, enforcing protocol compliance by rejecting any undefined integer values.

## Lifecycle & Ownership
- **Creation:** The enum constants, Adventure and Creative, are instantiated by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs only once.
- **Scope:** Application-level. The instances persist for the entire lifetime of the application. They are effectively global, static singletons.
- **Destruction:** The instances are garbage collected when the JVM shuts down. Manual memory management is not required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that cannot be changed after initialization. The static VALUES array is also populated once and is not intended for modification.
- **Thread Safety:** This class is inherently thread-safe. As immutable, statically-initialized objects, GameMode constants can be safely accessed and read from any thread without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation for serialization. |
| fromValue(int value) | GameMode | O(1) | **Critical:** Deserializes an integer into a GameMode instance. Throws ProtocolException if the value is not a valid enum identifier. |

## Integration Patterns

### Standard Usage
This enum is primarily used for setting or checking game state. The fromValue method is the designated entry point for converting raw network data into a type-safe object.

```java
// Deserializing a game mode from a network packet
int modeId = packet.readVarInt();
GameMode currentMode = GameMode.fromValue(modeId);

// Using the enum in game logic
if (currentMode == GameMode.Creative) {
    player.setCanFly(true);
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Value:** Never compare game modes using their integer values. This defeats the purpose of type safety and makes the code brittle.
  - **BAD:** `if (currentMode.getValue() == 0)`
  - **GOOD:** `if (currentMode == GameMode.Adventure)`
- **Invalid Deserialization:** Do not bypass the fromValue method when deserializing. The built-in bounds check is essential for protocol security.
  - **BAD:** `GameMode mode = GameMode.VALUES[unsafeValue]; // Throws ArrayIndexOutOfBoundsException`
  - **GOOD:** `GameMode mode = GameMode.fromValue(unsafeValue); // Throws a controlled ProtocolException`

## Data Pipeline
GameMode serves as a data model, not an active processing component. It represents a piece of state that flows through various systems.

> Flow:
> Server Game State -> **GameMode.getValue()** -> Network Serializer -> TCP Packet -> Client Network Deserializer -> **GameMode.fromValue()** -> Client Game State -> Game Logic System

