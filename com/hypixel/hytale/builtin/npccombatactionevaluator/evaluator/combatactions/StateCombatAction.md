---
description: Architectural reference for StateCombatAction
---

# StateCombatAction

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator.combatactions
**Type:** Transient Command Object

## Definition
```java
// Signature
public class StateCombatAction extends CombatActionOption {
```

## Architecture & Concepts
The StateCombatAction is a data-driven command object that represents a single, atomic action within an NPC's combat AI behavior tree: changing the NPC's state. It is a concrete implementation of the **Strategy Pattern**, where different CombatActionOption subclasses define various AI behaviors.

This class is not a service or a manager. It is a lightweight, declarative instruction instantiated by the Hytale Codec system from game asset files (e.g., JSON definitions for an NPC archetype). Its primary role is to bridge the declarative world of AI configuration with the imperative world of the server's Entity Component System (ECS).

During a game tick, the CombatActionEvaluator for a given NPC will select an appropriate CombatActionOption to execute. If a StateCombatAction is chosen, its execute method is invoked, which then queues a state change command into the ECS CommandBuffer for deferred processing.

## Lifecycle & Ownership
- **Creation:** StateCombatAction instances are created exclusively by the Hytale `BuilderCodec` system during server startup or when game assets are loaded. They are deserialized from NPC definition files. **Warning:** Manual instantiation of this class in game logic is a design violation.
- **Scope:** An instance of this class is effectively a read-only template. It is owned by the asset management system and shared by all NPC instances of the same archetype. Its lifetime is tied to the loaded game assets, not to any individual NPC entity.
- **Destruction:** Instances are eligible for garbage collection only when their corresponding NPC archetype assets are unloaded by the server.

## Internal State & Concurrency
- **State:** This object is **effectively immutable** after its creation via the codec. The internal fields `state` and `subState` are set once during deserialization and are not intended to be modified at runtime. It holds no entity-specific state.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutable nature. However, the execution context is critical. The `execute` method is **not re-entrant** and **must** be called from the main server thread that processes the ECS world. It manipulates a CommandBuffer, which is a mechanism designed to safely queue state changes for later application within the main game loop tick. Calling `execute` from an asynchronous task or worker thread will lead to state corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Queues a state change command. Must be called by the CombatActionEvaluator on the main server thread. |
| isBasicAttackAllowed(...) | boolean | O(1) | A behavioral flag. Always returns false, preventing the NPC from performing basic attacks while this action is considered active. |
| getState() | String | O(1) | Returns the target main state name. |
| getSubState() | String | O(1) | Returns the target sub-state name. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define it declaratively within an NPC's AI configuration file. The system handles its instantiation and execution.

The conceptual flow is managed by the CombatActionEvaluator:

```java
// PSEUDO-CODE: Illustrates the engine's internal usage
// This code does not exist in one place but represents the flow.

// 1. The evaluator selects this action from a list of possibilities
StateCombatAction chosenAction = (StateCombatAction) evaluator.selectNextAction();

// 2. The evaluator invokes the action on the target entity
chosenAction.execute(
    entityIndex,
    archetypeChunk,
    commandBuffer,
    npcRole,
    evaluator,
    valueStore
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new StateCombatAction()`. The object's properties must be injected via the `CODEC` system from data files. Direct instantiation bypasses the entire data-driven AI architecture.
- **State Mutation:** Do not attempt to modify the `state` or `subState` fields after the object has been created. These are considered read-only configuration data.
- **External Execution:** Do not call the `execute` method from outside the context of a `CombatActionEvaluator`. The evaluator manages critical pre-conditions and post-conditions, such as clearing action timeouts.

## Data Pipeline

The StateCombatAction acts as a control-flow component, not a data-transformation one. It translates a high-level AI intent into a low-level engine command.

> Flow:
> NPC AI Asset File -> Hytale Codec System -> **StateCombatAction Instance** -> CombatActionEvaluator Selection Logic -> `execute()` call -> ECS CommandBuffer -> Entity State Component Update

