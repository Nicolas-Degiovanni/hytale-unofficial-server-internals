---
description: Architectural reference for CombatActionEvaluator
---

# CombatActionEvaluator

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator
**Type:** Transient

## Definition
```java
// Signature
public class CombatActionEvaluator extends Evaluator<CombatActionOption> implements Component<EntityStore> {
```

## Architecture & Concepts

The CombatActionEvaluator is the central decision-making engine for an NPC's combat behavior. As a component within the server-side Entity Component System (ECS), it is attached to individual NPC entities to grant them autonomous combat capabilities. It does not control movement or animation directly; rather, it selects high-level combat strategies which are then translated into concrete actions by other systems.

This class implements the *Evaluator* pattern, a core design in Hytale's AI framework. Its primary function is to calculate a "utility score" for a predefined set of possible `CombatActionOption` assets. These options represent discrete abilities like special attacks, defensive maneuvers, or support skills. The evaluator selects the most advantageous option based on the current combat context, such as target distance, cooldowns, and entity state.

The system distinguishes between two primary types of attacks:
1.  **Basic Attacks:** A simple, often repeating sequence of attacks managed via `currentBasicAttackSet`. These are the NPC's default actions when no special abilities are available or viable.
2.  **Combat Actions:** More complex abilities defined by `CombatActionOption` assets. These are subject to rigorous utility calculation, cooldowns, and situational conditions.

The evaluator's logic is further segmented by the NPC's current "sub-state", allowing for different behaviors in different phases of a fight (e.g., an "enraged" state might unlock a different set of available actions).

## Lifecycle & Ownership

-   **Creation:** An instance of CombatActionEvaluator is created and configured by a higher-level NPC management system when an NPC entity is spawned or assigned a combat-capable `Role`. The constructor requires a `CombatActionEvaluatorConfig` asset, which defines the specific actions, conditions, and parameters for that NPC type. This process is part of the entity archetype instantiation pipeline.

-   **Scope:** The component's lifetime is strictly tied to the NPC entity it is attached to. It persists as long as the NPC is active in the world. Its internal state represents the real-time combat decisions and cooldowns for that specific NPC.

-   **Destruction:** The component is destroyed and its memory is reclaimed when the parent NPC entity is removed from the `EntityStore`. This typically occurs upon the NPC's death or when it is despawned from the world.

## Internal State & Concurrency

-   **State:** The CombatActionEvaluator is highly stateful and mutable. It maintains a significant amount of runtime state, including:
    *   The currently executing `CombatActionOption` (`currentAction`).
    *   The current target entity (`primaryTarget`).
    *   Cooldown timers for basic attacks (`basicAttackCooldown`) and special actions (`lastUsedNanos` within option holders).
    *   The sequence and state of basic attacks (`nextBasicAttackIndex`, `currentBasicAttackSet`).
    *   A pre-calculated map of available actions organized by combat sub-state (`optionsBySubState`).

-   **Thread Safety:** This component is **not thread-safe** and is designed to be operated on exclusively by the main server thread responsible for its parent entity's game logic updates. All modifications to game state (e.g., initiating an attack animation, applying damage) must be deferred via a `CommandBuffer`. Direct mutation from asynchronous tasks or other threads will lead to race conditions, data corruption, and server instability. The presence of `CommandBuffer` in method signatures is a strict enforcement of this pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| selectNextCombatAction(...) | void | O(N*M) | Evaluates all available actions for the current sub-state against all potential targets. Selects and executes the highest utility action. This is the primary entry point for the decision-making tick. |
| completeCurrentAction(...) | void | O(1) | Signals the successful completion of the current action, updating cooldowns and state. |
| terminateCurrentAction() | void | O(1) | Forcefully interrupts the current action without marking it as complete. Used for state changes or interruptions. |
| canUseBasicAttack(...) | boolean | O(1) | Checks if the NPC is in a state where a basic attack can be initiated. |
| tickBasicAttackCoolDown(dt) | void | O(1) | Advances the internal cooldown timer for basic attacks. Must be called every game tick. |
| hasTimedOut(dt) | boolean | O(1) | Checks if the current action has exceeded its maximum allowed execution time. |
| setupNPC(...) | void | O(N) | Initializes the component and its associated `CombatActionOption` assets for a specific NPC role or entity instance. |

## Integration Patterns

### Standard Usage

The CombatActionEvaluator is managed by a server-side ECS `System`, likely a `BehaviorSystem` or `AIUpdateSystem`. On each game tick for a given NPC, the system retrieves this component and orchestrates its operation.

```java
// Within a server-side System that processes NPC logic
ArchetypeChunk<EntityStore> chunk = ...;
CommandBuffer<EntityStore> buffer = ...;
int entityIndex = ...;

CombatActionEvaluator evaluator = chunk.getComponent(entityIndex, CombatActionEvaluator.getComponentType());
Role npcRole = chunk.getComponent(entityIndex, Role.getComponentType());
ValueStore valueStore = chunk.getComponent(entityIndex, ValueStore.getComponentType());

// Update internal timers every tick
evaluator.tickBasicAttackCoolDown(deltaTime);

// If no special action is running, attempt to select a new one
if (evaluator.getCurrentAction() == null) {
    evaluator.selectNextCombatAction(entityIndex, chunk, buffer, npcRole, valueStore);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new CombatActionEvaluator()`. The constructor has complex dependencies on configuration assets and role data that are only resolved correctly by the engine's NPC factory systems. Always retrieve the component from an entity.
-   **State Tampering:** Do not manually set fields like `currentAction` or `primaryTarget` from outside the class. Use the provided public methods (`completeCurrentAction`, `terminateCurrentAction`) to manage the component's state machine correctly.
-   **Multi-threaded Access:** Never call methods on this component from any thread other than the entity's primary update thread. Doing so will bypass the `CommandBuffer` and cause severe concurrency issues.
-   **Skipping Ticks:** Failing to call timer methods like `tickBasicAttackCoolDown` on every game update will cause cooldowns and timeouts to function incorrectly, breaking combat pacing.

## Data Pipeline

The CombatActionEvaluator functions as a stateful processor in the NPC AI data pipeline. It transforms high-level sensory data into a single, actionable combat decision.

> Flow:
> 1.  **Input:** External systems update an NPC's `TargetMemory` component with lists of known hostile and friendly entities.
> 2.  **Evaluation:** A controlling `System` invokes `selectNextCombatAction`.
> 3.  **Utility Calculation:** The evaluator iterates through its configured `CombatActionOption`s for the current state. For each option, it calculates a utility score against potential targets.
> 4.  **Selection:** Using the calculated scores and a `predictability` factor, it performs a weighted random selection to choose the best action. This prevents NPCs from becoming perfectly predictable.
> 5.  **Execution:** The `execute` method on the chosen `CombatActionOption` is called.
> 6.  **Output:** The `execute` method uses the `CommandBuffer` to queue changes to other components, such as triggering an animation, applying a status effect, or initiating movement towards a target.

