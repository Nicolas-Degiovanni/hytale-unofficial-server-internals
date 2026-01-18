---
description: Architectural reference for RecentSustainedDamageCondition
---

# RecentSustainedDamageCondition

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.conditions
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class RecentSustainedDamageCondition extends ScaledCurveCondition {
```

## Architecture & Concepts

The RecentSustainedDamageCondition is a key component within Hytale's server-side NPC Utility AI system. It does not implement complex logic itself; rather, it serves as a data provider for its parent class, ScaledCurveCondition. Its primary function is to quantify how much damage an NPC has recently sustained and feed that value into a configurable evaluation curve.

Architecturally, this class acts as a bridge between the Entity Component System (ECS) and the NPC decision-making framework. It queries the state of an NPC entity—specifically the DamageMemory component—and translates that state into a numerical input for the AI. The parent ScaledCurveCondition then uses this input to calculate a utility score, which the AI uses to weigh potential actions. For example, a high sustained damage value might result in a high utility score for a "Flee" or "Use Healing Potion" action.

This class is designed to be **stateless and data-driven**. Its behavior is defined entirely by external configuration (the evaluation curve in its parent) and the runtime state of the NPC entity it is evaluating.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via code using the new keyword. Instead, they are deserialized from NPC behavior definition assets (e.g., JSON files) by the engine's asset loader, utilizing the provided static CODEC field. This allows designers to declaratively build complex AI behaviors without writing code.
-   **Scope:** An instance of this condition persists as long as the parent NPC behavior asset is loaded in memory. It is effectively immutable after creation.
-   **Destruction:** The object is marked for garbage collection when the corresponding NPC behavior asset is unloaded by the server.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It holds no mutable fields and does not cache any data between calls. All information is read directly from the ArchetypeChunk and DamageMemory component during the evaluation of the getInput method.
-   **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature ensures that it can be evaluated by multiple threads in the server's NPC processing job system without risk of race conditions or data corruption. The atomicity of the operation is guaranteed by the underlying ECS framework.

## API Surface

The primary interaction with this class is through methods inherited from its parent and called by the NPC decision-making engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setupNPC(Holder) | void | O(1) | Part of the NPC initialization contract. Ensures the target NPC entity has a DamageMemory component attached. Throws an error if the component cannot be added. |
| getInput(...) | protected double | O(1) | **(Internal Engine API)**. Retrieves the recent damage value from the entity's DamageMemory component. This is the core data-providing method. |

## Integration Patterns

### Standard Usage

This condition is not intended to be used directly in procedural Java code. It is designed to be defined within an NPC's behavior asset file. The engine then instantiates and integrates it into the NPC's decision-making logic. The `setupNPC` method is called by the engine once when the NPC's AI is initialized.

```java
// Conceptual engine code during an AI evaluation tick
// This code is representative of how the engine uses the condition,
// not how a developer or designer would.

// For a given NPC, the engine iterates through its configured conditions
for (Condition condition : npc.getBehavior().getConditions()) {
    if (condition instanceof RecentSustainedDamageCondition) {
        // The engine calls the evaluation logic from the parent class,
        // which in turn calls this class's getInput method.
        double utilityScore = condition.evaluate(npc, context);
        
        // This score is then used to influence action selection
        actionSelector.addScore(associatedAction, utilityScore);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new RecentSustainedDamageCondition()`. The object will be unconfigured (lacking a curve from its parent) and will not be part of any AI system. All instances must be created via the asset loading pipeline.
-   **Missing Dependency:** Attempting to use this condition on an NPC entity that lacks the DamageMemory component will result in a runtime error. The `setupNPC` method is designed to prevent this, but it is a critical dependency that must be satisfied.

## Data Pipeline

The flow of data through this component is linear and unidirectional during an AI evaluation tick.

> Flow:
> External Damage Source -> Server Damage System -> **DamageMemory Component** (State is updated) -> NPC AI Evaluation Tick -> **RecentSustainedDamageCondition.getInput()** (Reads state) -> ScaledCurveCondition (Transforms value to utility score) -> Action Selector (Weighs decisions) -> Final NPC Action

