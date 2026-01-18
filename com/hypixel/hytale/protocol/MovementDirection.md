---
description: Architectural reference for MovementDirection
---

# MovementDirection

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum MovementDirection {
```

## Architecture & Concepts
MovementDirection is a foundational data type within the Hytale network protocol. It serves as a type-safe, immutable contract for representing discrete entity movement states. Its primary architectural function is to decouple the high-level game logic from the low-level integer values used for network serialization.

Instead of passing raw integers, which are prone to error and lack semantic meaning, the system uses instances of MovementDirection. This ensures that only valid movement states can be represented at compile time. The enum is responsible for the translation to and from the integer format required for efficient wire transmission, centralizing the serialization logic and making the rest of the codebase cleaner and more robust.

This component is critical for interpreting player input packets on the server and synchronizing entity movement across all clients.

### Lifecycle & Ownership
- **Creation:** All enum constants (None, Forward, Back, etc.) are instantiated by the Java Virtual Machine during class loading. This process is automatic, thread-safe, and occurs once at application startup. User code does not and cannot create new instances.
- **Scope:** As static final constants, all MovementDirection instances are global singletons that persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded only when the application's class loader is garbage collected, which typically coincides with the termination of the JVM process.

## Internal State & Concurrency
- **State:** MovementDirection is deeply immutable. Each enum constant has a final integer field, value, which is assigned at creation and can never be changed. The static VALUES array is also final and is used as a performance-optimized cache for deserialization lookups.
- **Thread Safety:** This class is unconditionally thread-safe. Its immutable nature guarantees that it can be safely read, passed, and shared across any number of threads without requiring any external synchronization or locks.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for network serialization. |
| fromValue(int value) | MovementDirection | O(1) | **Static factory.** Deserializes an integer into a MovementDirection instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets related to entity physics and player control.

**Serialization (Client -> Server):**
```java
// Game logic determines the direction
MovementDirection playerMove = MovementDirection.ForwardLeft;

// The network layer retrieves the integer for the packet
int valueToSend = playerMove.getValue();
packet.writeInt(valueToSend);
```

**Deserialization (Server <- Client):**
```java
// The network layer reads the integer from the packet
int receivedValue = packet.readInt();

// The value is converted back to a safe enum type for game logic
MovementDirection direction = MovementDirection.fromValue(receivedValue);
server.processPlayerMovement(player, direction);
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined and retrieved via `getValue()`. Relying on `ordinal()` is extremely fragile and will break the protocol if the declaration order of the enum constants is ever changed.
- **Manual Value Checking:** Do not write `if/else` or `switch` statements to manually map integers to directions. Always use the provided `fromValue` static factory, as it centralizes validation and error handling.

## Data Pipeline
MovementDirection is not a processing stage but rather the data payload itself as it flows between systems. It represents a transformation of data from a raw, untyped format to a structured, type-safe one.

> **Client-Side Flow:**
> Raw Keyboard Input -> Input System -> **MovementDirection.Forward** -> Network Serializer -> `1` (integer in packet) -> TCP/UDP Stack

> **Server-Side Flow:**
> TCP/UDP Stack -> `1` (integer in packet) -> Network Deserializer -> **MovementDirection.Forward** -> Game Logic -> Entity Physics Update

