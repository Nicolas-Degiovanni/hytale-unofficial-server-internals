---
description: Architectural reference for ScaledCurveCondition
---

# ScaledCurveCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions.base
**Type:** Component (Abstract Base)

## Definition
```java
// Signature
public abstract class ScaledCurveCondition extends Condition {
```

## Architecture & Concepts
The ScaledCurveCondition is a foundational abstract class within the server-side NPC Decision Maker framework. It serves as a template for any condition whose outcome can be mapped to a numerical utility score via a response curve. This component is a cornerstone of Hytale's **Utility AI** system, where NPCs select behaviors by scoring potential actions based on a set of conditions.

This class employs the **Template Method Pattern**. It provides the invariant algorithm for calculating utility:
1.  Obtain a raw numerical input from the current game state.
2.  Evaluate that input using a configurable ScaledResponseCurve.
3.  Return the resulting Y-value as the final utility score.

Subclasses are responsible for implementing the variant part of the algorithm: the `getInput` method. This design cleanly separates the generic curve-evaluation logic from the specific, domain-aware logic of gathering inputs (e.g., distance to target, current health percentage, or time of day).

Critically, this class and its derivatives are designed to be **data-driven**. The `ABSTRACT_CODEC` field indicates that concrete conditions are defined in external asset files, not hard-coded in Java. This allows game designers to create, combine, and tune complex NPC behaviors without requiring engine recompilation.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the `new` keyword. They are instantiated by the Hytale **Codec** system during server initialization or when behavior assets are loaded. The `ABSTRACT_CODEC` is responsible for deserializing asset data into a fully configured object, including its `responseCurve`.
-   **Scope:** A deserialized ScaledCurveCondition object is typically long-lived and shared. It persists as part of an NPC archetype's behavior definition for the entire server session. It is treated as a stateless, reusable component.
-   **Destruction:** Managed by the Java Garbage Collector. Instances are eligible for cleanup when the associated NPC archetypes or behavior assets are unloaded from memory. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state consists of the `responseCurve` field. This state is set once upon deserialization and is considered **effectively immutable** thereafter. The class itself is stateless with respect to evaluation; its methods do not modify its own fields.
-   **Thread Safety:** This class is **thread-safe** under the assumption that its `responseCurve` is not modified after creation. The `calculateUtility` method is a pure function of its inputs and internal immutable state. Multiple threads (e.g., parallel AI evaluation tasks for different NPCs) can safely invoke `calculateUtility` on a shared instance without locks or synchronization.

    **Warning:** Subclasses that introduce mutable state will break this thread-safety guarantee and may cause severe, difficult-to-diagnose concurrency bugs.

## API Surface
The public contract is focused on providing the utility score and metadata for the AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateUtility(...) | double | O(1) | The primary entry point. Executes the template method to return a utility score. Returns 0.0 if the input is Double.MAX_VALUE. |
| getInput(...) | double | O(C) | **Abstract.** Subclasses must implement this to provide the raw input value from the game state. Complexity is dependent on the subclass. |
| getSimplicity() | int | O(1) | Returns a constant value (30) used by the Decision Maker to potentially prioritize or optimize evaluation order. |
| getResponseCurve() | ScaledResponseCurve | O(1) | Returns the configured response curve asset. |

## Integration Patterns

### Standard Usage
A developer does not use this class directly. Instead, they create a concrete subclass that provides a specific input. The engine's Decision Maker system then invokes instances of that subclass.

```java
// 1. A concrete subclass is defined
public class HealthCondition extends ScaledCurveCondition {
    // This class would also have its own CODEC for deserialization

    @Override
    protected double getInput(int selfIndex, ArchetypeChunk<EntityStore> chunk, ...) {
        // Logic to get the entity's health component
        HealthComponent health = chunk.get(selfIndex, HealthComponent.class);
        return health.getCurrent() / (double) health.getMax(); // Returns a value from 0.0 to 1.0
    }
}

// 2. The engine's AI system uses it (conceptual)
// This logic is internal to the DecisionMaker service.
double utility = healthCondition.calculateUtility(npcEntityId, worldState, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate a subclass with `new`. The object will be incomplete, lacking a `responseCurve` which is injected by the codec system. This will result in a NullPointerException.
    ```java
    // ANTI-PATTERN: Bypasses the data-driven asset loading system
    HealthCondition condition = new HealthCondition(); 
    condition.calculateUtility(...); // Throws NullPointerException
    ```
-   **Stateful Subclasses:** Avoid adding mutable fields to subclasses. Condition instances are shared across many NPC evaluations and must remain stateless to ensure thread safety and predictable behavior.

## Data Pipeline
This component is a critical step in the NPC behavior evaluation pipeline. It transforms a specific, concrete game state metric into an abstract utility score that the AI system can use for comparison and decision-making.

> Flow:
> Game State Query -> **Subclass.getInput()** -> Raw Numerical Value -> **ScaledCurveCondition.calculateUtility()** -> ScaledResponseCurve Evaluation -> Final Utility Score (double) -> AI Decision Logic

---

