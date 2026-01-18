---
description: Architectural reference for PlayerSleep
---

# PlayerSleep

**Package:** com.hypixel.hytale.builtin.beds.sleep.components
**Type:** State Definition Interface

## Definition
```java
// Signature
public sealed interface PlayerSleep permits PlayerSleep.FullyAwake, PlayerSleep.MorningWakeUp, PlayerSleep.NoddingOff, PlayerSleep.Slumber {
    // Nested state definitions
}
```

## Architecture & Concepts
The PlayerSleep interface is a foundational component of the server-side sleep mechanic. It employs Java's sealed interface feature to define a finite, type-safe state machine for a player's sleep status. This is a modern implementation of the State pattern, ensuring that a player can only exist in one of the explicitly permitted states: FullyAwake, NoddingOff, Slumber, or MorningWakeUp.

This interface does not represent a service or a manager; it is a pure data contract that models the domain of player sleep. Each state is implemented as an immutable record or enum, capturing the necessary data for that specific state, such as the timestamp when the state was entered.

The primary consumer of this state machine is the PlayerSomnolence component, which holds the *current* PlayerSleep state for a given player entity. Server-side systems, such as a SleepSystem, query the PlayerSomnolence component and use pattern matching against the PlayerSleep interface to execute state-specific logic, such as accelerating time or waking the player.

### Lifecycle & Ownership
- **Creation:** Instances of PlayerSleep states are created exclusively through static factory methods on the nested record types (e.g., NoddingOff.createComponent). These factories are invoked by higher-level systems in response to game events, such as a player interacting with a bed. The factory methods immediately wrap the new state object within a PlayerSomnolence component, which is the intended container.
- **Scope:** A specific PlayerSleep state object is ephemeral and has a very short lifetime. It exists only as long as the player entity remains in that state. When a state transition occurs, the existing state object is discarded and replaced by a new one representing the new state.
- **Destruction:** Objects are managed by the Java Garbage Collector. An instance becomes eligible for collection as soon as the owning PlayerSomnolence component is updated with a new PlayerSleep state, removing all strong references to the old state object.

## Internal State & Concurrency
- **State:** The design is fundamentally immutable. The interface itself is stateless. The concrete implementations (FullyAwake, NoddingOff, etc.) are either enums or records with final fields. This immutability guarantees that a state, once created, cannot be altered, which simplifies reasoning about the system and eliminates a large class of potential bugs.
- **Thread Safety:** All implementations of PlayerSleep are inherently thread-safe due to their immutability. They can be safely read from any thread without synchronization. However, the process of *transitioning* a player from one state to another must be synchronized. This is typically handled by ensuring that all modifications to an entity's components, such as PlayerSomnolence, occur exclusively on the main server game loop thread.

## API Surface
The public contract is defined by the static factory methods within the nested state types. These are the sole entry points for creating new sleep states.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MorningWakeUp.createComponent(worldTime) | PlayerSomnolence | O(1) | Creates a component for a player waking up. Captures the current game time. |
| NoddingOff.createComponent() | PlayerSomnolence | O(1) | Creates a component for a player initiating sleep. Captures the current real-world time. |
| Slumber.createComponent(worldTime) | PlayerSomnolence | O(1) | Creates a component for a player in deep sleep. Captures the current game time. |

## Integration Patterns

### Standard Usage
The intended use is for a game system to pattern-match on an entity's PlayerSomnolence component to drive logic. State transitions are performed by calling a factory method and replacing the component on the entity.

```java
// A system processing a player entity
PlayerSomnolence somnolence = entity.get(PlayerSomnolence.class);

// Pattern matching to drive state-specific logic
switch (somnolence.state()) {
    case PlayerSleep.NoddingOff noddingOff -> {
        // Check if enough time has passed to transition to Slumber
        if (shouldTransitionToSlumber(noddingOff)) {
            WorldTimeResource time = context.getResource(WorldTimeResource.class);
            PlayerSomnolence newComponent = PlayerSleep.Slumber.createComponent(time);
            entity.set(newComponent); // Atomically replace the component
        }
    }
    case PlayerSleep.Slumber slumber -> {
        // Accelerate world time
    }
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate state records directly using their constructors (e.g., new PlayerSleep.NoddingOff(Instant.now())). This bypasses the factory methods which correctly wrap the state in the required PlayerSomnolence component, leading to system inconsistencies.
- **State Caching:** Do not hold a reference to a PlayerSleep state object outside the scope of a single system update or tick. The state is volatile and can be replaced at any moment. Always re-fetch the PlayerSomnolence component from the entity at the start of a processing block.

## Data Pipeline
PlayerSleep is not a data processing pipeline; it is a set of states within a larger state transition flow managed by server systems.

> Flow:
> Player Bed Interaction Event -> BedInteractionSystem -> **PlayerSleep.NoddingOff.createComponent()** -> PlayerSomnolence component attached to Entity -> SleepSystem (next tick) -> Detects NoddingOff, transitions to **Slumber** -> TimeSystem -> Detects Slumber, accelerates time -> Condition Met (e.g., daytime) -> SleepSystem -> Transitions to **MorningWakeUp** -> Player Control is restored.

