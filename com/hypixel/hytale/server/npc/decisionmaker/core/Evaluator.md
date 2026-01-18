---
description: Architectural reference for Evaluator
---

# Evaluator

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class Evaluator<OptionType extends Option> {
```

## Architecture & Concepts

The Evaluator is the central component of the server-side Utility AI system for Non-Player Characters (NPCs). It provides a generic, performance-oriented framework for selecting the most appropriate action, or Option, for an NPC to perform in a given situation. This class does not make decisions itself; rather, it orchestrates the evaluation of a pre-configured list of potential Options.

Its core design is based on the following principles:

*   **Utility Scoring:** Each potential action (an Option) is scored based on its "utility" in the current game context. The utility is a numerical value representing the desirability of an action. The action with the highest utility is typically chosen.
*   **Weighted Options:** Options are associated with a weight coefficient, allowing for a preliminary, high-performance culling of irrelevant choices before expensive utility calculations are performed. The `initialise` method pre-sorts all options by this weight, which is a critical optimization.
*   **Contextual Evaluation:** The `evaluate` method receives a comprehensive `EvaluationContext` and an `ArchetypeChunk` representing the current state of the NPC and its environment. This allows for highly contextual and dynamic decision-making.
*   **Predictability:** The system includes a `predictability` factor. A value of 1.0 results in deterministic behavior (always choosing the highest-scoring option), while lower values introduce weighted randomness, making NPC behavior less robotic and predictable.

The Evaluator is an abstract base class. Concrete implementations are created for different categories of decisions, such as combat targeting, ability selection, or idle behaviors.

### Lifecycle & Ownership

-   **Creation:** A concrete Evaluator instance is typically created and configured once as part of an NPC's `Role` definition, which acts as a template for all NPCs of that type. It is not instantiated per-NPC.
-   **Scope:** The Evaluator's lifetime is tied to the `Role` it belongs to. It persists as long as that NPC type is active in the game world, serving evaluations for thousands of individual NPC entities.
-   **Destruction:** The object is eligible for garbage collection when its parent `Role` is unloaded, for instance, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency

-   **State:** The primary internal state is the `options` list, which contains all possible `OptionHolder` instances. This list is mutable during the `initialise` phase but must be treated as **immutable** thereafter. The nested `OptionHolder` class contains a mutable `utility` field, which is updated on every call to `evaluate`. This field is transient and specific to a single evaluation cycle.

-   **Thread Safety:** This class is **not thread-safe**. The `evaluate` method modifies the `utility` state within the shared `OptionHolder` objects. Concurrent calls to `evaluate` on the same Evaluator instance from different threads will produce race conditions and lead to highly unpredictable and erroneous AI behavior.

    **WARNING:** All evaluations for a given Evaluator instance must be serialized. The engine's architecture ensures this by processing NPC updates within a single-threaded tick or by using a pool of Evaluators where each instance is confined to a single worker thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialise() | void | O(N log N) | Pre-sorts and prepares the internal options list. Must be called once before any evaluations. |
| setupNPC(Role) | void | O(N) | Propagates NPC-type-specific setup to all contained Options. |
| evaluate(...) | OptionHolder | O(N) | The core evaluation method. Iterates through options, calculates utility, and selects the best one. |

## Integration Patterns

### Standard Usage

The Evaluator is designed to be used by a higher-level AI or behavior system. The system retrieves the appropriate Evaluator for an NPC and invokes it during the entity's update tick.

```java
// Within an NPC behavior system update loop...

// 1. Get the appropriate evaluator for this NPC's role.
Evaluator<MyCombatOption> combatEvaluator = npcRole.getCombatEvaluator();

// 2. Create a context object with current world state.
EvaluationContext context = new EvaluationContext(world, npc);

// 3. Execute the evaluation to get the best action.
Evaluator.OptionHolder bestActionHolder = combatEvaluator.evaluate(
    npc.getIndexInChunk(),
    npc.getArchetypeChunk(),
    commandBuffer,
    context
);

// 4. Execute the action associated with the winning option.
if (bestActionHolder != null) {
    MyCombatOption chosenAction = bestActionHolder.getOption();
    chosenAction.execute(npc, commandBuffer);
}
```

### Anti-Patterns (Do NOT do this)

-   **Skipping Initialization:** Failure to call `initialise()` before `evaluate()` will result in incorrect behavior. The weight-based culling optimization depends on the pre-sorted list of options.
-   **Post-Initialization Modification:** Do not add, remove, or reorder the `options` list after `initialise()` has been called. This will violate the class's internal assumptions and lead to undefined behavior.
-   **Concurrent Evaluation:** Never share a single Evaluator instance across multiple threads that are simultaneously evaluating different NPCs. This will cause a race condition on the internal `utility` state of the `OptionHolder` objects.

## Data Pipeline

The flow of data through the Evaluator is a classic "Select and Act" AI pattern.

> Flow:
> NPC Update Tick -> **Evaluator.evaluate(context)** -> Utility Calculation per Option -> Highest Utility Option Selected -> **Return OptionHolder** -> AI System Executes Action

