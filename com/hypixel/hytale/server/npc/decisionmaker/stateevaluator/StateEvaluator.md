---
description: Architectural reference for StateEvaluator
---

# StateEvaluator

**Package:** com.hypixel.hytale.server.npc.decisionmaker.stateevaluator
**Type:** Component (Data-Oriented)

## Definition
```java
// Signature
public class StateEvaluator extends Evaluator<StateOption> implements Component<EntityStore> {
```

## Architecture & Concepts

The StateEvaluator is a fundamental component within Hytale's server-side NPC Artificial Intelligence framework. It operates as a data-holding component within the server's Entity Component System (ECS), attaching AI decision-making logic to a specific entity.

Its primary architectural purpose is to drive a **Utility-Based AI** model. In this model, an NPC is presented with a list of possible actions or states, known as StateOptions. The StateEvaluator, when triggered by a governing system, calculates a numerical "utility" score for each option based on the current world state, entity state, and environmental factors. The option with the highest score is considered the most desirable action for the NPC to take.

This component is entirely data-driven. Its behavior, such as the frequency of evaluation, cooldowns after state changes, and the list of possible states, is configured via external asset files. The static CODEC field is responsible for deserializing this data into a functional StateEvaluator instance at runtime. This design decouples the high-level AI behavior (defined in data) from the low-level evaluation logic (defined in code), allowing designers to tune NPC behavior without modifying the core engine.

## Lifecycle & Ownership

-   **Creation:** A StateEvaluator is never instantiated directly using its constructor. It is created by the engine's serialization system when an entity is spawned from an archetype that includes this component. The static `CODEC` field dictates how to build the object from asset data.
-   **Scope:** The lifecycle of a StateEvaluator instance is strictly bound to the entity to which it is attached. It persists as long as the parent entity exists within the game world.
-   **Destruction:** The component is marked for garbage collection and its resources are released when its parent entity is removed from the `EntityStore`. This process is managed automatically by the ECS framework.

## Internal State & Concurrency

-   **State:** This component is highly **mutable**. It maintains several internal state variables that change during the game loop, most notably `timeUntilNextExecute`, which acts as a timer to throttle evaluation frequency. Its configuration fields, such as `executeFrequency` and `rawOptions`, are loaded once and are treated as immutable for the component's lifetime.
-   **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by the main server world-tick thread. Its methods manipulate internal state without any locks or synchronization primitives. Any concurrent access from other threads (e.g., networking or async tasks) will result in severe race conditions, corrupted state, and unpredictable AI behavior. All interactions must be marshaled to the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type identifier for this component, used for ECS queries. |
| prepareOptions(helper) | void | O(N) | **Critical Initialization.** Converts raw option data into a processed, indexed format for efficient evaluation. Must be called once before the evaluator is used. |
| shouldExecute(interval) | boolean | O(1) | Core throttling mechanism. Returns true if the internal timer has elapsed, indicating an evaluation should run. |
| prepareEvaluationContext(context) | void | O(1) | Populates a shared EvaluationContext object with this evaluator's specific parameters before a utility calculation pass. |
| onStateSwitched() | void | O(1) | Resets the evaluation timer to the `stateChangeCooldown`. This must be called by the owning system after a successful state transition to prevent state-flapping. |
| setActive(boolean) | void | O(1) | Enables or disables the evaluator. An inactive evaluator will always return false from `shouldExecute`. |

## Integration Patterns

### Standard Usage

The StateEvaluator is managed by a higher-level AI system, such as a `DecisionMakerSystem`. This system queries for all entities possessing this component and executes the evaluation logic as part of the main server tick.

```java
// Simplified example within an AI System's update loop
for (ArchetypeChunk<EntityStore> chunk : query) {
    ComponentAccessor<StateEvaluator> evaluators = chunk.getAccessor(StateEvaluator.getComponentType());

    for (int i = 0; i < chunk.size(); i++) {
        StateEvaluator evaluator = evaluators.get(i);
        if (evaluator.isActive() && evaluator.shouldExecute(deltaTime)) {
            // Prepare context and run the evaluation
            EvaluationContext context = evaluator.getEvaluationContext();
            evaluator.prepareEvaluationContext(context);

            // The parent Evaluator class provides the core 'evaluate' method
            StateOption bestOption = evaluator.evaluate(i, chunk, commandBuffer, context);

            if (bestOption != null) {
                // Command the entity to switch to the new state
                // ...
                evaluator.onStateSwitched(); // CRITICAL: Signal the cooldown
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new StateEvaluator()`. The component is incomplete without being configured and created by the engine's asset `CODEC`.
-   **Ignoring Cooldowns:** Failure to call `onStateSwitched` after a state change will cause the NPC to re-evaluate its state on the very next tick, potentially leading to rapid, undesirable oscillations between states.
-   **External State Modification:** Do not modify fields like `timeUntilNextExecute` or `minimumUtility` at runtime. These are intended to be configured via assets. For dynamic behavior, use other components or systems to influence the `EvaluationContext`.

## Data Pipeline

The flow of data begins with asset configuration and proceeds through the live evaluation loop on the server.

> Flow:
> NPC Asset File -> Engine Asset Loader -> **StateEvaluator CODEC** -> Instantiated **StateEvaluator** Component -> AI System Tick -> `shouldExecute` check -> `evaluate` method -> Highest Utility `StateOption` -> State Transition Command -> Entity State Updated

