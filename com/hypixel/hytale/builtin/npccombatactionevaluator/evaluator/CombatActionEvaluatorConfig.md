---
description: Architectural reference for CombatActionEvaluatorConfig
---

# CombatActionEvaluatorConfig

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator
**Type:** Configuration Model / DTO

## Definition
```java
// Signature
public class CombatActionEvaluatorConfig {
```

## Architecture & Concepts

The CombatActionEvaluatorConfig class is a pure data container that defines the complete behavioral profile for an NPC's combat decision-making. It is not an active component; rather, it serves as a static, deserialized blueprint that is consumed by the active CombatActionEvaluator system.

This class is fundamentally tied to Hytale's asset and codec system. The static final field CODEC, an instance of BuilderCodec, acts as both a schema definition and a deserializer. Game designers and modders define NPC combat behavior in external data files (e.g., JSON), and this codec is responsible for parsing that data, validating it, and instantiating a CombatActionEvaluatorConfig object.

The design promotes a data-driven approach to AI. Instead of hard-coding combat logic, behavior is defined by composing and configuring several key elements:
*   **Available Actions:** A master dictionary of all possible combat abilities an NPC can use.
*   **Action Sets:** Subsets of the available actions, grouped for use in specific combat states (e.g., an "enraged" state might use a different set of actions than a "defensive" state).
*   **Run Conditions:** A gating mechanism that determines *if* the AI should even re-evaluate its current action. This prevents constant, expensive AI calculations every tick.
*   **Utility Thresholds:** Numeric cutoffs (MinRunUtility, MinActionUtility) that integrate with a utility-based AI model. Actions are scored, and only those exceeding the threshold are considered viable candidates for execution.

This object, once loaded, is treated as immutable and is often shared between multiple instances of the same NPC type to conserve memory.

## Lifecycle & Ownership

-   **Creation:** Instances are not created via a direct constructor call (e.g., using new). They are exclusively instantiated by the Hytale asset loading pipeline. The static CODEC field is invoked by an asset manager to deserialize an NPC definition file from disk into a memory-resident Java object.
-   **Scope:** An instance of CombatActionEvaluatorConfig has a scope tied to its corresponding asset. It is loaded once and cached, persisting in memory for as long as the NPC type it defines is needed. It may live for the entire server session.
-   **Destruction:** The object is eligible for garbage collection only when its defining asset is unloaded from the AssetManager. This typically occurs during a server shutdown or a major world change that purges unused assets.

## Internal State & Concurrency

-   **State:** The object's state is **effectively immutable**. All fields are populated once during deserialization by the BuilderCodec. The public API exposes only getters, and there are no methods for mutation. Collections returned by getters are either unmodifiable or should be treated as read-only.
-   **Thread Safety:** This class is **thread-safe for reads**. Due to its immutable nature, a single cached instance can be safely accessed by multiple AI controllers on different threads simultaneously without requiring locks or other synchronization primitives. This is critical for performance in a server environment with many active NPCs.

## API Surface

The public contract is composed entirely of accessors for reading the deserialized configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAvailableActions() | Map | O(1) | Returns the master map of action names to their definitions. |
| getActionSets() | Map | O(1) | Returns the mapping of combat states to their corresponding ActionSet configurations. |
| getRunConditions() | String[] | O(1) | Returns the array of condition assets that gate the evaluator's execution. |
| getMinRunUtility() | double | O(1) | Returns the minimum utility score required for the RunConditions to pass. |
| getMinActionUtility() | double | O(1) | Returns the minimum utility score required for any single combat action to be chosen. |
| getPredictabilityRange() | double[] | O(1) | Returns the min/max range used to inject randomness into AI decision-making. |

## Integration Patterns

### Standard Usage

This object is not used directly. It is a dependency consumed by higher-level AI systems. A component attached to an NPC entity would hold a reference to this config, which is then passed to the evaluator.

```java
// Conceptual example within an AI controller
CombatActionEvaluatorConfig config = npc.getComponent(AIConfigComponent.class).getCombatConfig();
CombatActionEvaluator evaluator = context.getService(CombatActionEvaluator.class);

// The evaluator uses the config to make a decision
CombatAction chosenAction = evaluator.evaluate(npc, target, config);
if (chosenAction != null) {
    npc.executeAction(chosenAction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new CombatActionEvaluatorConfig()`. This will result in a partially initialized object with null fields, bypassing all validation logic defined in the codec and causing runtime NullPointerExceptions in the AI systems.
-   **State Mutation:** Do not attempt to modify the collections returned by the getters. Modifying this shared configuration at runtime will cause unpredictable and non-deterministic behavior across all NPCs of the same type. This can lead to difficult-to-diagnose AI bugs.

## Data Pipeline

The data for this object originates from static asset files and flows into the real-time AI decision loop.

> Flow:
> NPC Definition File (JSON) -> Server Asset Loader -> **CombatActionEvaluatorConfig.CODEC** -> In-Memory Config Instance -> CombatActionEvaluator -> AI Decision

