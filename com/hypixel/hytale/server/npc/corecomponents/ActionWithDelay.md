---
description: Architectural reference for ActionWithDelay
---

# ActionWithDelay

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Component (Abstract Base Class)

## Definition
```java
// Signature
public abstract class ActionWithDelay extends ActionBase {
```

## Architecture & Concepts
ActionWithDelay is an abstract foundational component within the server-side NPC (Non-Player Character) AI framework. It does not represent a complete, executable action but rather provides a stateful, non-blocking delay mechanism for concrete subclasses to inherit.

Its primary architectural role is to solve the problem of pausing an NPC's behavior without halting the server's main processing thread. Instead of using a blocking sleep, this class implements a state-based timer. An NPC's controlling logic, such as a Behavior Tree or Finite State Machine, will instantiate a subclass of ActionWithDelay. The controlling logic then drives the timer forward each server tick by calling the processDelay method.

The core design separates the configuration of a delay from its execution:
1.  **Preparation:** The `prepareDelay` method calculates a random duration based on a pre-configured range. This allows the system to determine *how long* to wait without immediately starting the timer.
2.  **Activation:** The `startDelay` method transitions the component into its "delaying" state, beginning the countdown.
3.  **Processing:** The game engine's main update loop provides a delta-time value to the `processDelay` method, which decrements the timer and reports when the delay has concluded.

This component is a primitive building block for creating more complex and natural-seeming AI behaviors, such as an NPC waiting for a moment before speaking or pausing between patrols.

### Lifecycle & Ownership
-   **Creation:** An instance is created as part of a concrete subclass's instantiation (e.g., `ActionLookAround`). These subclasses are typically constructed by a builder system (as evidenced by the `BuilderActionWithDelay` constructor parameter) when an NPC's behavior profile is loaded or generated.
-   **Scope:** The object's lifetime is tied directly to the NPC's current action. It is a transient, stateful object that exists only as long as the NPC is performing the specific behavior it represents. It is not shared between NPCs.
-   **Destruction:** The instance is marked for garbage collection when the NPC's AI controller discards the action and transitions to a new state. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is highly mutable and exists specifically to manage time-based state. Its internal fields `delay` and `isDelaying` are modified frequently during the component's lifecycle. The initial `delayRange` is considered immutable after construction.
-   **Thread Safety:** **This class is not thread-safe.** It is designed with the strict assumption that all method calls for a given instance will originate from a single thread, typically the server's main tick thread responsible for updating NPC logic. Unsynchronized, concurrent access from multiple threads will lead to race conditions and unpredictable timer behavior. Do not share instances of this class or its subclasses across threads without external locking.

## API Surface
The public and protected methods form the contract for subclasses and the AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| processDelay(float dt) | boolean | O(1) | The primary update method. Decrements the timer if active. Returns true if the delay is complete or was never started, false otherwise. |
| isDelaying() | boolean | O(1) | Checks if the component is currently in its active countdown phase. |
| isDelayPrepared() | boolean | O(1) | Checks if a delay duration has been calculated via `prepareDelay`. |
| prepareDelay() | void | O(1) | Calculates a random delay duration from the configured range and stores it internally. Does not start the timer. |
| clearDelay() | void | O(1) | Resets all internal timer state to its default, non-delaying values. |
| startDelay(EntitySupport support) | void | O(1) | Activates the countdown. Registers the action with the entity's support system and sets the internal state to "delaying". |

## Integration Patterns

### Standard Usage
A concrete subclass will typically call `prepareDelay` during its own initialization or activation phase. The `startDelay` method is then called to begin the timer. The engine is responsible for calling `processDelay` on each update tick.

```java
// In a hypothetical concrete subclass: ActionPause
@Override
public void onActionStarted(EntitySupport support) {
    // 1. Calculate how long to wait
    prepareDelay();

    // 2. If the duration is valid, start the countdown
    if (isDelayPrepared()) {
        startDelay(support);
    }
}

// The NPC engine calls this every tick
@Override
public boolean process(float dt) {
    // 3. The engine drives the timer. If processDelay returns true,
    // the delay is over and the action can complete.
    if (processDelay(dt)) {
        // Delay finished, do next thing...
        return true; // Signal completion
    }
    return false; // Signal still busy
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Ticking:** Do not call `processDelay` from your own loops. This method is designed to be driven exclusively by the server's global update tick to ensure timing is consistent with the rest of the game world.
-   **State Mismanagement:** Calling `startDelay` without first calling `prepareDelay` will result in a zero-duration delay, which is likely an error. Always prepare the delay before starting it.
-   **Instance Re-use:** Do not re-use an `ActionWithDelay` instance for a different NPC or even for a different execution of the same action on the same NPC. These objects are stateful and should be treated as single-use.

## Data Pipeline
This component does not process a data stream in a traditional sense. Instead, it acts as a stateful gate within the NPC's logic flow. Its "pipeline" is a control flow driven by the game loop.

> Flow:
> NPC Behavior Controller -> `prepareDelay()` -> `startDelay()` -> **ActionWithDelay** <- (Game Loop Tick `dt`) -> `processDelay()` -> Boolean "is_finished" -> NPC Behavior Controller

