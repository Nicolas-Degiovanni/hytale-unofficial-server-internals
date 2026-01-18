---
description: Architectural reference for MovementType
---

# MovementType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum MovementType {
```

## Architecture & Concepts
The MovementType enum is a foundational data type within the Hytale network protocol. It serves as a type-safe, canonical representation of all possible character locomotion states. Its primary architectural role is to translate high-level gameplay concepts, such as a player walking or swimming, into a low-level, compact integer format suitable for efficient network serialization.

This class is a critical component of the data contract between the client and the server. By providing static factory methods like *fromValue*, it acts as a validation and deserialization gateway. Any attempt to decode a network packet containing an undefined movement state will result in a ProtocolException, immediately halting the processing of the invalid data and preventing state corruption further down the line. The pre-cached VALUES array is a performance optimization to avoid the overhead of repeated calls to the standard `values()` method, which allocates a new array on each invocation.

### Lifecycle & Ownership
- **Creation:** All enum constants are instantiated once by the JVM during class loading. This process is automatic and occurs before any application code can reference the class.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the JVM process. They are effectively permanent, global singletons.
- **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual memory management or destruction process.

## Internal State & Concurrency
- **State:** Inherently immutable. Each enum constant is a singleton instance whose internal integer *value* is declared final and set at creation time. Its state cannot be modified post-initialization.
- **Thread Safety:** This class is unconditionally thread-safe. As immutable singletons, its instances can be safely shared and read across any number of threads without requiring locks or other synchronization primitives. The static methods are pure functions and are also safe for concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the movement state for network serialization. |
| fromValue(int value) | MovementType | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of player state packets.

**Serialization (Client to Server)**
```java
// Example: Preparing a player state update to be sent over the network
PlayerStatePacket packet = new PlayerStatePacket();
MovementType currentMovement = determinePlayerMovement(); // Returns MovementType.Running
packet.setMovement(currentMovement.getValue());
networkManager.send(packet);
```

**Deserialization (Server from Client)**
```java
// Example: Processing an incoming player state packet on the server
void handlePlayerState(PlayerStatePacket packet) {
    try {
        MovementType type = MovementType.fromValue(packet.getMovement());
        // Update game logic with the validated movement type
        player.setMovementState(type);
    } catch (ProtocolException e) {
        // Disconnect the client for sending invalid data
        disconnectClient(packet.getSource(), "Invalid movement type");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The protocol relies on the explicit integer *value* assigned in the constructor. Using ordinals is brittle and will break the network contract if the declaration order of the enum constants is ever changed.
- **Ignoring Exceptions:** Never call *fromValue* without a try-catch block when processing untrusted network input. An unhandled ProtocolException can crash the processing thread.
- **Comparison by Value:** Do not compare movement types by their integer values (e.g., `if (type.getValue() == 4)`). Always compare the enum instances directly for type safety and readability (e.g., `if (type == MovementType.Running)`).

## Data Pipeline
MovementType acts as a translation layer at the boundary of the network stack and the game logic engine.

> **Outbound Flow (Serialization):**
> Game Logic State -> **MovementType.Running** -> `getValue()` -> Integer (4) -> Packet Serializer -> Network Buffer

> **Inbound Flow (Deserialization):**
> Network Buffer -> Packet Deserializer -> Integer (4) -> `fromValue(4)` -> **MovementType.Running** -> Game Logic State

