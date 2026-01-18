---
description: Architectural reference for WorldSleep
---

# WorldSleep

**Package:** com.hypixel.hytale.builtin.beds.sleep.resources
**Type:** State Model

## Definition
```java
// Signature
public sealed interface WorldSleep permits WorldSleep.Awake, WorldSlumber {
```

## Architecture & Concepts
The WorldSleep interface is a core state model representing the collective sleep status of the game world. It employs Java's sealed interface feature to create a closed, type-safe state machine. This design pattern is critical for ensuring that the world's sleep state can only ever be one of a small, well-defined set of possibilities, enforced at compile time.

The primary role of this interface is to eliminate "primitive obsession" and invalid states. Instead of representing the world's sleep status with a boolean, integer, or string, the engine uses the WorldSleep type system. This prevents logic errors where an invalid state could be introduced. The two permitted implementations are:

*   **WorldSleep.Awake:** Represents the default state where time progresses normally and players are active.
*   **WorldSlumber:** Represents the state where one or more players are in bed, potentially accelerating the day-night cycle.

This component is not a service or manager; it is a value type used to communicate state between different game systems, such as the PlayerInteraction system, the TimeOfDayManager, and the server-wide GameState broadcaster.

## Lifecycle & Ownership
- **Creation:** Instances of WorldSleep are not created dynamically in the traditional sense. The `Awake` state is a static singleton enum, loaded once by the JVM. The `WorldSlumber` state (not shown) is likely instantiated by a central `SleepManager` when the first player enters a bed.
- **Scope:** These state objects are effectively immortal and globally accessible constants or short-lived value objects. They do not hold references to external resources and are cheap to create and garbage collect.
- **Destruction:** The `Awake.INSTANCE` singleton persists for the lifetime of the application. Other state objects like `WorldSlumber` are eligible for garbage collection as soon as they are no longer referenced by the authoritative world state manager.

## Internal State & Concurrency
- **State:** Implementations of WorldSleep are designed to be immutable. The `Awake` enum is inherently immutable. Any other implementation, such as `WorldSlumber`, should also be designed as an immutable data carrier.
- **Thread Safety:** This model is unconditionally thread-safe. As immutable value types, instances can be safely passed between threads without locks or synchronization. The system that *manages* the current WorldSleep state (e.g., a `WorldStateManager`) is responsible for its own thread safety, typically through atomic references or synchronized state transitions.

## API Surface
The primary API is the type system itself. The interface defines no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Awake.INSTANCE | WorldSleep.Awake | O(1) | Singleton constant representing the world's awake state. |

## Integration Patterns

### Standard Usage
This type is used to pattern match game logic, typically within a central game loop or state update handler. Modern Java `switch` expressions are the preferred mechanism for handling the different states.

```java
// In a hypothetical WorldStateManager
WorldSleep currentSleepState = this.getWorldSleepState();

// Use a switch expression to safely handle all possible states
String statusMessage = switch (currentSleepState) {
    case WorldSleep.Awake awake -> "The world is awake.";
    case WorldSlumber slumber -> "Players are sleeping, time is accelerating.";
    // No default is needed due to the sealed interface
};

eventBus.post(new WorldStatusUpdateEvent(statusMessage));
```

### Anti-Patterns (Do NOT do this)
- **Null Checks:** Do not use `null` to represent an uninitialized or unknown sleep state. The type system is designed to make all states explicit. A `null` value indicates a severe logic error elsewhere in the engine.
- **Instanceof Chain:** While functional, using a chain of `if (state instanceof ...)` is less expressive and more error-prone than using a `switch` expression, which guarantees exhaustive checking of all permitted types.
- **Extending the Interface:** Do not attempt to implement the WorldSleep interface outside of its designated package and module. The `sealed` and `permits` keywords explicitly forbid this, and any attempt will result in a compilation error.

## Data Pipeline
WorldSleep is not a data processor but a data payload. It represents the outcome of processing other events.

> Flow:
> PlayerUseBedEvent -> **SleepManager** -> (State Transition) -> New **WorldSleep** instance -> WorldStateUpdate -> EventBus -> TimeOfDayManager / Client Notifier

