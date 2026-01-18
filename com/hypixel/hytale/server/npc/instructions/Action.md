---
description: Architectural reference for the Action interface, the core contract for NPC behavior execution.
---

# Action

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Action extends RoleStateChange, IAnnotatedComponent, IComponentExecutionControl {
    // ... methods
}
```

## Architecture & Concepts
The Action interface is the fundamental contract for defining a single, discrete, and executable behavior for a server-side Non-Player Character (NPC). It represents one unit of work within the NPC's broader AI logic, such as navigating to a point, engaging a target, or interacting with an object.

This interface is a core component of the server's AI state machine and behavior tree systems. It is designed to be implemented by concrete classes (e.g., a hypothetical WalkToAction or AttackAction) that encapsulate the specific logic for that behavior. An Action is not a long-running process but rather a stateful object whose `execute` method is typically invoked once per server tick by a higher-level AI controller, such as a Role.

By inheriting from `RoleStateChange`, `IAnnotatedComponent`, and `IComponentExecutionControl`, the Action contract is deeply integrated into the NPC component system. This structure allows Actions to be dynamically configured, inspected, and managed by the server, enabling complex and data-driven AI behaviors.

### Lifecycle & Ownership
- **Creation:** Concrete Action implementations are not created directly. They are instantiated by AI configuration loaders or behavior factories when an NPC's Role is defined or assigned. They are part of a pre-defined set of potential behaviors for a given NPC type.
- **Scope:** An Action object's lifetime is typically tied to its parent Role or AI controller. It persists as a template for a behavior. A specific *execution* of an Action, however, is ephemeral, managed by the `activate` and `deactivate` lifecycle methods.
- **Destruction:** The Action object is garbage collected when the parent NPC or its associated Role is unloaded from the world.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, all non-trivial implementations of Action are expected to be **highly stateful**. For example, a `WalkToAction` must store its target destination and current pathfinding progress. This internal state is initialized in `activate` and should be cleared in `deactivate`.
- **Thread Safety:** **Not thread-safe.** All methods on the Action interface are intended to be called exclusively from the main server thread during the NPC update phase of the game loop. The `execute` method directly mutates the world state via the passed `Ref<EntityStore>`. Calling any method from an asynchronous thread will lead to severe concurrency violations, data corruption, and server instability.

## API Surface
The public contract of Action defines a clear execution lifecycle for a single NPC behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(N) | Pre-condition check. Determines if the Action is valid to run in the current world state. Often involves world queries. |
| execute(...) | boolean | O(N) | Executes one tick of the Action's logic. Returns true if the Action is complete, false otherwise. |
| activate(...) | void | O(1) | Initializes the Action's internal state for a new execution. Called once before the first call to `execute`. |
| deactivate(...) | void | O(1) | Cleans up the Action's internal state after execution finishes or is interrupted. Called once after the final `execute`. |
| isActivated() | boolean | O(1) | Returns true if the Action is currently between an `activate` and `deactivate` call. |

## Integration Patterns

### Standard Usage
The canonical usage pattern involves an AI controller iterating through a list of available Actions for an NPC. The controller selects a high-priority Action where `canExecute` returns true, then manages its lifecycle across multiple server ticks.

```java
// Simplified example from a hypothetical AI controller
Action currentAction = getNextAction(); // Logic to select an action

if (currentAction != null && !currentAction.isActivated()) {
    // Initialize the action for execution
    currentAction.activate(npcRole, infoProvider);
}

if (currentAction != null && currentAction.isActivated()) {
    // Execute one tick of the action's logic
    boolean isFinished = currentAction.execute(entityRef, npcRole, infoProvider, deltaTime, worldStore);

    if (isFinished) {
        // Clean up the action and nullify the reference
        currentAction.deactivate(npcRole, infoProvider);
        currentAction = null;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Leakage:** Do not implement an Action where state from one execution can affect the next. All transient state must be reset within the `activate` and `deactivate` methods. Failure to do so will result in unpredictable and buggy NPC behavior.
- **Skipping Lifecycle Methods:** Never call `execute` without a preceding call to `activate`. This will bypass critical state initialization and likely result in a NullPointerException or invalid state calculations. Similarly, always call `deactivate` when an action is completed or interrupted.
- **Direct Instantiation:** Concrete Action classes should not be created with `new` in general gameplay code. They are part of an NPC's definition and should be managed by the AI configuration system.

## Data Pipeline
The Action interface serves as a processor in the NPC AI control flow rather than a traditional data pipeline. It consumes world state and produces entity state changes.

> **Flow:**
> AI Decision Engine (e.g., Behavior Tree) -> Selects **Action** -> `activate()` -> `execute()` (per tick) -> Mutates `EntityStore` -> `deactivate()` -> AI Decision Engine selects next Action

