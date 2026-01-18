---
description: Architectural reference for ParticleCollisionAction
---

# ParticleCollisionAction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Type / Enumeration

## Definition
```java
// Signature
public enum ParticleCollisionAction {
```

## Architecture & Concepts
The ParticleCollisionAction enum defines a constrained, type-safe set of behaviors that a particle can exhibit upon collision with another object in the game world. Its primary role is to serve as a data contract within the network protocol, translating high-level gameplay concepts into a low-level, efficient integer representation for serialization.

This component is fundamental to the data integrity of the particle system. By providing a strict mapping between integers and named constants, it prevents the propagation of invalid or undefined states that could arise from malformed network packets. The static factory method, fromValue, acts as a validation gateway during deserialization, ensuring that any data read from a network stream conforms to one of the known, valid actions.

## Lifecycle & Ownership
- **Creation:** All instances (Expire, LastFrame, Linger) are compile-time constants. They are instantiated once by the JVM when the ParticleCollisionAction class is loaded into memory.
- **Scope:** Application-wide static constants. These instances persist for the entire lifetime of the application.
- **Destruction:** The instances are garbage collected only when the application's ClassLoader is unloaded, which typically occurs at shutdown. There is no manual memory management.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final integer value that cannot be changed after initialization. The collection of all constants, VALUES, is also a static final array, preventing runtime modification.
- **Thread Safety:** **Fully Thread-Safe**. As immutable singletons, instances of ParticleCollisionAction can be read and passed between threads without any need for external synchronization or locking mechanisms. All methods are re-entrant and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the network-serialized integer representation of the enum constant. |
| fromValue(int value) | static ParticleCollisionAction | O(1) | Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the integer is out of the defined range. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of game state, particularly for particle effect packets. Logic should always operate on the enum type and only convert to or from an integer at the protocol boundary.

```java
// Deserialization from a network buffer
int actionId = buffer.readVarInt();
ParticleCollisionAction action = ParticleCollisionAction.fromValue(actionId);

// Game logic based on the action
switch (action) {
    case Expire:
        particle.markForRemoval();
        break;
    case Linger:
        particle.setPhysicsEnabled(false);
        break;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Magic Numbers:** Avoid using raw integer literals (0, 1, 2) directly in game logic. This creates brittle code that is difficult to read and maintain. Always reference the named constants like ParticleCollisionAction.Expire.

    ```java
    // BAD: Unreadable and error-prone
    if (particle.getCollisionAction() == 0) {
        // ...
    }

    // GOOD: Clear, type-safe, and self-documenting
    if (particle.getCollisionAction() == ParticleCollisionAction.Expire) {
        // ...
    }
    ```

- **Unsafe Deserialization:** Never assume an integer from an external source is valid. Bypassing the fromValue method by using the VALUES array directly can lead to an ArrayIndexOutOfBoundsException and crash the client or server.

    ```java
    // DANGEROUS: Bypasses bounds checking
    ParticleCollisionAction action = ParticleCollisionAction.VALUES[actionId];

    // CORRECT: Handles invalid data gracefully
    try {
        ParticleCollisionAction action = ParticleCollisionAction.fromValue(actionId);
    } catch (ProtocolException e) {
        // Log error, disconnect client, or handle corrupted packet
    }
    ```

## Data Pipeline
ParticleCollisionAction serves as a data model, not an active processor. It represents a piece of data as it flows between the server's game simulation and the client's rendering engine.

> **Serialization Flow (Server -> Client):**
> Game Logic State -> **ParticleCollisionAction.Expire** -> getValue() -> `int 0` -> Network Packet

> **Deserialization Flow (Client <- Server):**
> Network Packet -> `int 0` -> fromValue(0) -> **ParticleCollisionAction.Expire** -> Particle Rendering System

