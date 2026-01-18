---
description: Architectural reference for ActionToggleStateEvaluator
---

# ActionToggleStateEvaluator

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine
**Type:** Behavioral Object

## Definition
```java
// Signature
public class ActionToggleStateEvaluator extends ActionBase {
```

## Architecture & Concepts
The ActionToggleStateEvaluator is a concrete implementation of the Command Pattern, designed to operate within the server-side NPC State Machine framework. Its sole responsibility is to modify the active status of a separate `StateEvaluator` component on a given NPC entity.

This class acts as a control mechanism, decoupling the high-level logic of a state machine from the low-level implementation of an NPC's decision-making process. For example, when an NPC's state machine transitions from *Patrolling* to *InCombat*, an instance of ActionToggleStateEvaluator can be executed to **enable** a combat-specific `StateEvaluator`. This allows the NPC to begin evaluating threats and targets. Conversely, when combat ends, another instance can **disable** it, conserving computational resources.

It is a fundamental building block for creating dynamic and context-aware NPC behaviors without embedding complex conditional logic directly within the state machine graph.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via code. They are instantiated by the NPC behavior system during the server's asset loading phase. The system parses declarative configuration files (e.g., JSON definitions for an NPC's behavior tree) and uses the corresponding `BuilderActionToggleStateEvaluator` to construct the action object.
- **Scope:** The object is effectively a stateless template. A single instance is created for a given state machine definition and is reused for every execution of that action. Its lifetime is tied to the loaded NPC behavior asset, not to an individual NPC entity.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC behavior assets, such as during a zone unload or a server shutdown.

## Internal State & Concurrency
- **State:** Immutable. The primary state, a boolean flag indicating whether to enable or disable the target component, is declared as `final` and set once during construction. The object itself does not maintain any mutable state across executions.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. However, its `execute` method is a mutating operation on an external `StateEvaluator` component. The overall system relies on the guarantee that all actions for a single NPC entity are executed serially on a single thread, typically the main world-tick thread.

**WARNING:** Invoking the `execute` method concurrently for the same entity from multiple threads is not a supported operation and will result in a race condition, leading to unpredictable behavior in the target `StateEvaluator`.

## API Surface
The public contract is minimal, consisting only of the inherited `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Toggles the active state of the `StateEvaluator` component on the entity. Always returns true. Throws an AssertionError if the entity lacks a `StateEvaluator`. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be configured declaratively within an NPC's behavior definition and executed automatically by the state machine runtime.

The following conceptual example illustrates how the engine invokes the action.

```java
// CONCEPTUAL: This code is executed by the NPC State Machine Engine.

// Assume 'currentAction' is a pre-loaded ActionToggleStateEvaluator
// instance and 'npcContext' holds all relevant entity data.
boolean wasSuccessful = currentAction.execute(
    npcContext.getEntityRef(),
    npcContext.getRole(),
    npcContext.getSensorInfo(),
    deltaTime,
    npcContext.getComponentStore()
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionToggleStateEvaluator()`. The framework's asset loader is responsible for creating these objects from configuration via their dedicated builder. Manual creation bypasses the intended declarative design.
- **Entity Misconfiguration:** Attaching this action to a state transition for an NPC that does not possess a `StateEvaluator` component will result in a runtime `AssertionError`. Ensure the target entity's component architecture is valid.
- **Stateful Extension:** Do not extend this class to add complex logic or internal state. Its purpose is to be a simple, stateless command. For more complex operations, a different `ActionBase` implementation should be created.

## Data Pipeline
ActionToggleStateEvaluator functions as a control-flow trigger, not a data processor. Its execution initiates a state change in a downstream component.

> Flow:
> NPC State Machine Transition → **ActionToggleStateEvaluator.execute()** → Entity Component Store Lookup → `StateEvaluator.setActive(boolean)` → Modified NPC Decision-Making Logic

