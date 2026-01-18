---
description: Architectural reference for WeightedAction
---

# WeightedAction

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Transient Component

## Definition
```java
// Signature
public class WeightedAction extends AnnotatedComponentBase implements Action {
```

## Architecture & Concepts

The WeightedAction class is a structural component within the server-side NPC AI framework. It functions as a **Decorator**, wrapping a concrete Action implementation and associating a numerical *weight* with it. This pattern is fundamental to creating complex, non-deterministic AI behaviors.

Its primary role is to serve as a data-rich container for higher-level decision-making components, such as an ActionSelector or a behavior tree node. These orchestrators gather a collection of WeightedAction instances and use their respective weights to perform a probabilistic selection. For example, an NPC might have three possible idle behaviors, each wrapped in a WeightedAction. An action with a weight of 50.0 is significantly more likely to be chosen than one with a weight of 5.0.

This class is a pure pass-through for the entire Action interface contract. It adds no logic to the execution, activation, or lifecycle of the underlying action; it merely attaches metadata (the weight) used for selection *before* the action is ever executed.

## Lifecycle & Ownership

-   **Creation:** WeightedAction instances are not created directly via code. They are instantiated by the NPC asset loading pipeline, specifically by a BuilderSupport instance processing a BuilderWeightedAction. This process is driven by declarative NPC configuration files (e.g., JSON or HOCON assets).
-   **Scope:** The lifetime of a WeightedAction is strictly tied to its owning AI component, typically a Role or a specific behavior definition. It persists as long as the NPC's configured AI logic is loaded.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC or its AI configuration is unloaded or replaced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** The internal state of WeightedAction itself is **immutable**. The *action* and *weight* fields are final and assigned at construction. However, the wrapped Action object it delegates to is almost certainly stateful. The WeightedAction is a stateless decorator around a stateful target.

-   **Thread Safety:** This class contains no internal synchronization mechanisms. All calls are delegated directly to the wrapped Action. Therefore, its thread safety is entirely dependent on the implementation of the wrapped Action.

    **Warning:** It is assumed that all interactions with this class and the underlying Action occur on the main server thread during the entity update tick. Unsynchronized access from other threads will lead to race conditions and unpredictable AI behavior.

## API Surface

The primary purpose of this class is to expose the weight and delegate all Action-related responsibilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWeight() | double | O(1) | Returns the configured weight for this action. This is the sole unique contribution of the class. |
| (Action Interface) | (Varies) | (Varies) | Implements the full Action interface. All calls are directly forwarded to the wrapped Action instance. |

## Integration Patterns

### Standard Usage

This component is not intended for direct, imperative use. It is a declarative building block. A higher-level system consumes a list of these to make a decision.

```java
// Hypothetical AI controller logic
public Action selectNextAction(List<WeightedAction> possibleActions) {
    double totalWeight = possibleActions.stream()
        .mapToDouble(WeightedAction::getWeight)
        .sum();

    double choice = Math.random() * totalWeight;
    double cumulativeWeight = 0.0;

    for (WeightedAction weightedAction : possibleActions) {
        cumulativeWeight += weightedAction.getWeight();
        if (cumulativeWeight >= choice) {
            // The wrapped action is retrieved for execution
            return weightedAction.action;
        }
    }
    return null; // Or a default action
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new WeightedAction()`. This bypasses the asset pipeline and will result in an improperly configured component that is not integrated into the NPC's AI systems. All instances must be created from NPC asset definitions.
-   **State Modification:** Do not attempt to use reflection or other means to modify the internal *action* or *weight* fields after construction. The system relies on this data being static for the lifetime of the component.

## Data Pipeline

WeightedAction acts as a data point in a decision-making flow rather than a data processing pipeline.

> **Decision Flow:**
> NPC Behavior Controller -> Gathers list of potential `WeightedAction`s -> **Evaluates `getWeight()` from each** -> Probabilistic Selector chooses one -> The underlying `Action` is executed

