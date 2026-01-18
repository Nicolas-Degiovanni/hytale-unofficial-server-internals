---
description: Architectural reference for Timer
---

# Timer

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient

## Definition
```java
// Signature
public class Timer implements Tickable {
```

## Architecture & Concepts
The Timer class is a fundamental, stateful utility for implementing time-based logic within the server's game loop. It functions as a high-precision, tick-driven countdown mechanism, primarily designed to orchestrate NPC behaviors, ability cooldowns, and other scheduled game events without blocking the main server thread.

By implementing the Tickable interface, a Timer instance signals its intent to be updated on every cycle of the game loop. A higher-level system, such as an NPC's behavior manager, is responsible for creating and ticking the Timer. The Timer encapsulates all countdown logic, including support for repeating intervals, randomized start times, and pausing. Its state-machine design (RUNNING, PAUSED, STOPPED) provides a clean and predictable contract for consumers to query its status and react accordingly.

This component is a low-level building block. It does not emit events or trigger callbacks on its own. The owning system is responsible for polling the Timer's state, typically checking if it has stopped, to trigger subsequent actions.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new Timer()`) by a component that requires time-based logic. For example, an NPC's `AttackBehavior` might create a Timer to manage its attack cooldown.
- **Scope:** The lifecycle of a Timer is strictly bound to its owner. It persists only as long as the owning object holds a reference to it. It is designed for short-to-medium term operations within a specific logical context.
- **Destruction:** The object is eligible for garbage collection once its owner is destroyed and all references to it are released. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The Timer is a highly mutable object. Its core state consists of its current countdown `value` and its `TimerState`. These fields are constantly updated by the `tick` method and other state-transition calls like `pause` or `stop`. It does not cache any external data.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It is designed to be created, modified, and ticked exclusively by the main server thread. The `tick` method performs non-atomic read-modify-write operations on the `value` field. Calling methods like `pause`, `addValue`, or `restart` from a separate thread while the game loop is ticking this Timer will result in race conditions, undefined behavior, and server instability. All interactions with a Timer instance must be synchronized with the game loop.

## API Surface
The public API provides a contract for configuring, controlling, and querying the countdown state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start(...) | void | O(1) | Initializes and starts the timer with a full set of parameters, including randomized start values. This is the primary entry point for activation. |
| tick(float dt) | void | O(1) | The core update method. Decrements the internal value based on delta time. Must be called every game loop cycle for the timer to function. |
| stop() | void | O(1) | Immediately halts the timer and resets its value to zero. |
| pause() | void | O(1) | Freezes the timer at its current value. The timer will not progress until `resume` is called. |
| resume() | void | O(1) | Unpauses a paused timer, allowing it to continue its countdown. |
| restart() | void | O(1) | Resets the timer's value to a new randomized value between `minRestartValue` and `maxValue` and sets its state to RUNNING. |
| isRunning() | boolean | O(1) | Returns true if the timer is currently active and counting down. |
| isStopped() | boolean | O(1) | Returns true if the timer has completed its countdown or has been explicitly stopped. This is the primary method for polling completion. |

## Integration Patterns

### Standard Usage
The canonical use case involves an owning object creating a Timer, starting it, and then ticking it on every frame. The owner polls the Timer's state to determine when to execute an action.

```java
// An NPC behavior class that uses a Timer for a cooldown
public class CooldownBehavior implements Tickable {
    private final Timer cooldownTimer = new Timer();

    public void startCooldown() {
        // Start a 5-second cooldown (rate of 1.0 is 1 per second)
        // Parameters: minStart, maxStart, minRestart, maxRestart, rate, repeating
        cooldownTimer.start(5.0, 5.0, 5.0, 5.0, 1.0, false);
    }

    @Override
    public void tick(float dt) {
        // The owner is responsible for ticking its timer
        cooldownTimer.tick(dt);

        if (cooldownTimer.isStopped()) {
            // The cooldown has finished. The NPC can now perform the action again.
            // ... trigger action logic ...
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instances:** Do not share a single Timer instance across multiple, unrelated logical components. Each distinct cooldown or timed event should have its own dedicated Timer instance to avoid state corruption.
- **Asynchronous Modification:** Never call methods like `pause`, `stop`, or `addValue` from a different thread than the one calling `tick`. This will lead to severe concurrency issues.
- **Forgetting to Tick:** Instantiating and starting a Timer is not sufficient. If its `tick` method is not called consistently by the game loop, its internal value will never change, and it will never complete.

## Data Pipeline
The Timer does not process a data pipeline in the traditional sense. Instead, it operates on a control flow driven by the game engine's clock.

> Flow:
> Game Loop Tick -> Owner.tick(dt) -> **Timer.tick(dt)** -> Internal state `value` is decremented -> Owner polls `isStopped()` -> Game Action Triggered

