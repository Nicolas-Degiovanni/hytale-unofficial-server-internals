---
description: Architectural reference for MovementState
---

# MovementState

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum MovementState implements Supplier<String> {
```

## Architecture & Concepts
The MovementState enum provides a fixed, type-safe vocabulary for all possible locomotion states of a Non-Player Character (NPC) within the server environment. Its primary architectural role is to eliminate "stringly-typed" programming and magic constants, ensuring that different systems—such as AI, physics, and animation—can communicate NPC state changes reliably and without ambiguity.

By representing states as distinct, compile-time constants (e.g., JUMPING, WALKING), the system guarantees that only valid states can be assigned. This prevents a large class of runtime errors that could occur if states were represented by raw strings or integers.

The implementation of the Supplier interface, providing a human-readable string via the get method, suggests its use in logging, debugging, serialization, or network contexts where a string representation is required without sacrificing the type safety of the core logic.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. They are effectively static, singleton instances created once when the MovementState class is first referenced.
- **Scope:** The instances persist for the entire lifetime of the application. They are globally accessible as static final fields (e.g., MovementState.RUNNING).
- **Destruction:** The enum constants are garbage collected only when the defining class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** MovementState is **immutable**. Each enum constant holds a private final String field for its name, which is assigned at creation and can never be changed.
- **Thread Safety:** This class is inherently **thread-safe**. As immutable singletons, its constants can be read and passed between any number of threads without requiring locks or any other synchronization mechanisms.

## API Surface
The primary API consists of the predefined enum constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| JUMPING, FLYING, etc. | MovementState | O(1) | A public, static, final instance representing a discrete NPC movement state. |
| get() | String | O(1) | Returns the human-readable name of the state (e.g., "Jumping"). Fulfills the Supplier contract. |

## Integration Patterns

### Standard Usage
MovementState is intended to be used for state assignment and comparison within control flow structures like switch statements.

```java
// How a developer should normally use this
import com.hypixel.hytale.server.npc.movement.MovementState;

public void updateNpcState(Npc npc, MovementState newState) {
    npc.setCurrentState(newState);

    // Use in a switch for state-dependent logic
    switch (newState) {
        case JUMPING:
            // Trigger jump animation and apply vertical force
            break;
        case SPRINTING:
            // Increase movement speed and stamina drain
            break;
        case IDLE:
            // Play idle animation loop
            break;
        default:
            // Handle other states
            break;
    }

    // Log the new state using the Supplier interface
    log.info("NPC state changed to: " + newState.get());
}
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Never use the string representation for logical comparisons. Enum constants should be compared by reference using the equality operator (==), which is both faster and safer.
    - **BAD:** `if (state.get().equals("Jumping")) { ... }`
    - **GOOD:** `if (state == MovementState.JUMPING) { ... }`
- **Reliance on Ordinal:** Do not use the `ordinal()` method to persist or identify states. The integer value is fragile and will change if the declaration order of the enum constants is modified.
- **Direct Instantiation:** The Java language prevents direct instantiation of enums with the `new` keyword. There is no way to create instances other than the predefined constants.

## Data Pipeline
MovementState acts as a data model payload, not a processing stage. It is created by one system and consumed by others to trigger behavior changes.

> **Flow 1: AI Decision**
> AI Behavior Tree -> Decides to run -> Sets NPC state to **MovementState.RUNNING** -> Physics System -> Updates velocity calculation

> **Flow 2: Physics Event**
> Physics Engine -> Detects sustained airtime without upward velocity -> Sets NPC state to **MovementState.FALLING** -> Animation System -> Blends to falling animation

> **Flow 3: Network Replication**
> Server NPC State Manager -> Serializes **MovementState.WALKING** to network packet -> Client Packet Handler -> Deserializes state -> Client-side Animator -> Plays walking animation for proxy entity

