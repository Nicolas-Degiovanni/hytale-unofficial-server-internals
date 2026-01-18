---
description: Architectural reference for AbilityCombatAction
---

# AbilityCombatAction

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator.combatactions
**Type:** Configuration Object

## Definition
```java
// Signature
public class AbilityCombatAction extends CombatActionOption {
```

## Architecture & Concepts

The AbilityCombatAction class is a data-driven configuration object that defines a specific combat maneuver for a Non-Player Character (NPC). It is not an active, stateful service but rather a blueprint for an action, deserialized from server-side asset files. Each instance represents a single, discrete option within an NPC's behavioral repertoire, such as a sword swing, a spell cast, or a special move.

Architecturally, this class serves as a critical bridge between the high-level AI decision-making layer, represented by the CombatActionEvaluator, and the low-level server mechanics of the Interaction system. When an NPC's AI decides to perform an attack, it selects an appropriate AbilityCombatAction instance and invokes its `execute` method.

The execution logic follows two primary paths:
1.  **Self-Targeted Actions:** For abilities cast on the NPC itself, the class directly constructs and queues an InteractionChain with the server's InteractionManager. This is a fire-and-forget operation.
2.  **Externally-Targeted Actions:** For abilities targeting another entity, the class does not execute the interaction directly. Instead, it acts as a configuration provider for the CombatActionEvaluator. It populates a shared ValueStore (a blackboard system) with critical parameters like attack range and desired positioning. It then delegates control back to the evaluator, which manages the complex, multi-tick process of moving the NPC into position before finally triggering the interaction.

This delegation pattern cleanly separates the static *definition* of an action (AbilityCombatAction) from the dynamic *execution* and state management of that action (CombatActionEvaluator).

### Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically using the `new` keyword. They are deserialized from NPC behavior asset files (e.g., JSON definitions) by the server's asset loading pipeline. The static `CODEC` field defines the schema for this deserialization, mapping configuration keys to the object's fields.
-   **Scope:** An instance persists for as long as the corresponding NPC archetype is loaded in server memory. These objects are effectively immutable after being loaded and are treated as shared, static configuration data.
-   **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup when the server unloads the associated NPC archetypes, typically during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency
-   **State:** The object's state is considered immutable after its initial creation via the `CODEC`. It is a plain data holder for configuration properties like ability name, attack range, and required inventory slots. A single field, `maxRangeSquared`, is a derived value cached at creation time to optimize distance calculations.
-   **Thread Safety:** The object is inherently thread-safe for read operations due to its immutable nature. However, its methods, particularly `execute`, are designed to be called exclusively from the main server thread within the Entity Component System (ECS) update cycle. The `execute` method mutates entity state via a `CommandBuffer`, a standard ECS pattern for queueing modifications to be applied safely at the end of a tick.

    **Warning:** Calling `execute` from any thread other than the main server thread will bypass engine-level concurrency controls and lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Configures the CombatActionEvaluator or directly queues an InteractionChain. Initiates complex, potentially long-running downstream AI and combat processes. |
| isBasicAttackAllowed(...) | boolean | O(1) | A predicate used by the AI to determine if a default attack is permissible. Returns false if the target is within this ability's effective range, encouraging the use of this specific action over a generic one. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by most developers. It is a component of the NPC AI system, and its `execute` method is invoked by the CombatActionEvaluator. The evaluator selects an action from a list based on situational parameters and then executes it.

```java
// Conceptual example from within a CombatActionEvaluator
// An evaluator holds a list of potential actions for an NPC.

// 1. The evaluator selects the best AbilityCombatAction based on game state.
AbilityCombatAction chosenAction = findBestActionForSituation(npc, target);

// 2. The evaluator invokes the action, passing the necessary ECS context.
if (chosenAction != null) {
    chosenAction.execute(
        entityIndex,
        archetypeChunk,
        commandBuffer,
        npcRole,
        this, // The evaluator instance
        valueStore
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AbilityCombatAction()`. The object will be uninitialized and lack the critical configuration loaded from assets, causing NullPointerExceptions and undefined behavior. Always define actions in the appropriate server data files.
-   **State Mutation:** Do not attempt to modify the fields of an AbilityCombatAction instance after it has been loaded. The system treats these objects as immutable, and runtime modifications can lead to inconsistent behavior across different NPCs using the same configuration.
-   **External Execution:** Do not call the `execute` method from outside the CombatActionEvaluator or a similar AI-system context. Doing so bypasses the state machine that manages action timeouts, positioning, and targeting, resulting in broken NPC behavior.

## Data Pipeline
The flow of data and control for a targeted ability demonstrates the class's role as a configuration provider for the broader AI system.

> Flow:
> 1. Server Asset File (JSON) -> Asset Loader with `CODEC`
> 2. -> **AbilityCombatAction** Instance (In Memory)
> 3. -> CombatActionEvaluator selects this instance
> 4. -> `execute()` is called
> 5. -> **AbilityCombatAction** writes range and positioning goals to the AI ValueStore
> 6. -> CombatActionEvaluator reads ValueStore and controls NPC Movement System
> 7. -> Once in position, CombatActionEvaluator triggers the Interaction via the InteractionManager

