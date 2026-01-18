---
description: Architectural reference for ActionRandom
---

# ActionRandom

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Stateful Component

## Definition
```java
// Signature
public class ActionRandom extends ActionBase {
```

## Architecture & Concepts
The ActionRandom class is a fundamental composite node within the server-side NPC Behavior system. It functions as a **Weighted Random Selector**, designed to introduce non-deterministic behavior into an NPC's decision-making process.

Architecturally, it sits within a behavior tree or a similar state machine structure, acting as a parent to a collection of other actions. Its primary responsibility is not to perform a world-altering action itself, but to select one of its child actions to execute based on predefined weights. This allows for more dynamic and less predictable NPC behavior, such as an animal randomly choosing between grazing, wandering, or resting.

Unlike a priority-based selector that always tries actions in a fixed order, ActionRandom evaluates all potential child actions and then makes a probabilistic choice among the ones that are currently valid.

## Lifecycle & Ownership
- **Creation:** ActionRandom instances are not created directly. They are instantiated by the NPC asset loading system via a corresponding builder, specifically BuilderActionRandom. This process occurs when an NPC's behavior configuration is loaded from asset files into memory.
- **Scope:** An instance of ActionRandom is scoped to a single NPC's Role component. It persists for the entire lifetime of that NPC entity in the world.
- **Destruction:** The object is eligible for garbage collection when the parent NPC entity is removed from the world. The overridden lifecycle methods, such as `unloaded` and `removed`, ensure that all child actions are also properly notified to release their own resources.

## Internal State & Concurrency
- **State:** This component is highly **mutable** and stateful. It maintains state across multiple server ticks.
    - The `current` field tracks the sub-action that was chosen and is currently executing.
    - The `availableActions` array and `availableActionsCount` are transient state, rebuilt each time a new selection is needed.
    - This statefulness is essential for long-running child actions that may take several ticks to complete.

- **Thread Safety:** **CRITICAL: This class is not thread-safe.** Its internal state, particularly the `current` and `availableActionsCount` fields, is accessed and modified without any synchronization mechanisms. All method calls on a given instance must be confined to the single thread responsible for updating the associated NPC, which is typically the main server world-tick thread. Concurrent access will lead to state corruption and unpredictable behavior.

## API Surface
The public API is defined by its parent, ActionBase, and is intended for consumption by the NPC Role and behavior schedulers, not by general game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(N) | Checks if the component can run. Iterates through all N child actions to build a list of valid candidates for execution. |
| execute(...) | boolean | O(N) / O(1) | On the first call for a new selection, performs a weighted random choice in O(N) time. On subsequent calls, it delegates to the current child action in O(1) time. |
| loaded(Role) | void | O(N) | Propagates the `loaded` lifecycle event to all N child actions. |
| spawned(Role) | void | O(N) | Propagates the `spawned` lifecycle event to all N child actions. |
| unloaded(Role) | void | O(N) | Propagates the `unloaded` lifecycle event to all N child actions. |

## Integration Patterns

### Standard Usage
ActionRandom is not intended to be invoked directly. It is configured within an NPC asset file and managed by the NPC's Role. The engine's behavior scheduler drives its execution.

A conceptual game tick for an NPC with this component would look like this:
```java
// This code is conceptual and executed by the NPC's Role/Scheduler

// 1. The scheduler first checks if the action can run
boolean canRun = randomAction.canExecute(ref, role, sensorInfo, dt, store);

// 2. If it can, the scheduler executes it
if (canRun) {
    boolean isFinished = randomAction.execute(ref, role, sensorInfo, dt, store);
    if (isFinished) {
        // The action (and its chosen sub-action) is complete.
        // The scheduler can now consider other actions.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionRandom()`. The object is complex and requires its state to be injected by the asset pipeline via its corresponding Builder. Direct instantiation will result in a non-functional component.
- **State Manipulation:** Do not externally modify the `current` field. The internal logic for selecting, executing, and clearing the running sub-action is delicate. Forcibly changing this state will break the execution flow.
- **Multi-threaded Execution:** Never call `canExecute` or `execute` from multiple threads. This will cause race conditions during the selection of the `current` action and corrupt the `availableActions` list.

## Data Pipeline
The flow of control during a single decision-making cycle is a two-phase process managed by the AI scheduler.

> **Phase 1: Pre-computation (canExecute)**
> AI Scheduler Tick -> `ActionRandom.canExecute()` -> Iterates all child `WeightedAction.canExecute()` -> Populates internal `availableActions` array -> Returns `true` if any children are available

> **Phase 2: Execution (execute)**
> AI Scheduler Tick -> `ActionRandom.execute()` -> Performs weighted random selection on `availableActions` -> Sets internal `current` action -> Calls `current.activate()` -> Calls `current.execute()` -> Returns result from child

