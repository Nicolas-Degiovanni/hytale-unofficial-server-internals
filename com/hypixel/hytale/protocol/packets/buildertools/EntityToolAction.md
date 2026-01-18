---
description: Architectural reference for EntityToolAction
---

# EntityToolAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Enumeration

## Definition
```java
// Signature
public enum EntityToolAction {
   Remove(0),
   Clone(1),
   Freeze(2);
```

## Architecture & Concepts
The EntityToolAction enum provides a type-safe, compile-time constant representation for actions performed by in-game builder tools on entities. Its primary role is to serve as a contract between the client and server for network communication, ensuring that both endpoints agree on a fixed set of possible operations.

This class is a fundamental component of the game's network protocol layer. It translates a low-level integer, suitable for efficient network transport, into a high-level, self-documenting object used by the game logic. This pattern prevents the use of "magic numbers" in the codebase, improving readability and reducing the risk of bugs related to protocol mismatches. The inclusion of a `fromValue` factory method with bounds checking is a critical defensive programming measure against malformed or malicious packets.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during class loading. They are effectively static singletons. The internal VALUES array is also populated at this time.
- **Scope:** Application-wide. These constants persist for the entire lifetime of the JVM process.
- **Destruction:** The enum and its constants are garbage collected only when the class loader itself is unloaded, which typically occurs at application shutdown. There is no manual destruction.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that cannot be changed after instantiation. The class itself holds no mutable state.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of EntityToolAction can be safely shared and accessed across any number of threads without requiring synchronization primitives. The static factory method `fromValue` is also fully thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the action for network serialization. |
| fromValue(int value) | EntityToolAction | O(1) | **[Deserialization]** Converts a network integer back to its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used during packet serialization and deserialization. Game logic should operate on the enum constants directly, converting to and from integers only at the network boundary.

**Serialization (Sending a packet):**
```java
// Game logic determines an action
EntityToolAction action = EntityToolAction.Clone;

// The packet serializer retrieves the integer value for the network stream
int valueToSend = action.getValue(); // Results in 1
packet.writeInt(valueToSend);
```

**Deserialization (Receiving a packet):**
```java
// The packet deserializer reads an integer from the network stream
int receivedValue = packet.readInt();

// Convert the integer back to a safe type before passing to game logic
EntityToolAction action = EntityToolAction.fromValue(receivedValue);

// Game logic can now safely use the enum
switch (action) {
    case Clone:
        // handle clone logic
        break;
    case Remove:
        // handle remove logic
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Do not use the integer value for logical comparisons within the game code. This defeats the purpose of type safety and re-introduces magic numbers.

    ```java
    // BAD: Brittle and hard to read
    if (action.getValue() == 1) { ... }

    // GOOD: Type-safe and self-documenting
    if (action == EntityToolAction.Clone) { ... }
    ```
- **Ignoring Exceptions:** The ProtocolException thrown by `fromValue` is a critical security and stability feature. It indicates a malformed packet or a protocol version mismatch. It must not be ignored.

## Data Pipeline
EntityToolAction acts as a serialization and deserialization gateway for a specific data field within a larger network packet.

> **Outbound Flow (Serialization):**
> Game Logic (e.g., Player Input) -> **EntityToolAction.Freeze** -> `getValue()` -> `2` (int) -> Network Packet Buffer -> Server

> **Inbound Flow (Deserialization):**
> Client -> Network Packet Buffer -> `2` (int) -> `fromValue(2)` -> **EntityToolAction.Freeze** -> Game Logic (e.g., Entity System)

