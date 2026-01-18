---
description: Architectural reference for the Condition class, the foundation of the NPC Utility AI system.
---

# Condition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions.base
**Type:** Asset / Data-Driven Component

## Definition
```java
// Signature
public abstract class Condition implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, Condition>> {
```

## Architecture & Concepts

The abstract Condition class is the foundational component of the server-side NPC Utility AI decision-making engine. It represents a single, evaluatable criterion that an NPC uses to determine the "utility" or desirability of a potential action. This system moves beyond simple true/false logic, allowing for nuanced AI behavior where actions are scored on a continuous spectrum.

Conditions are not hard-coded logic; they are data-driven assets loaded from JSON files. This design allows game designers to define, combine, and tune complex AI behaviors without modifying engine source code. Each concrete implementation of Condition, such as TargetDistanceCondition or TimeOfDayCondition, encapsulates a specific query against the game state.

The system employs a polymorphic deserialization pattern managed by the static CODEC field. When the AssetStore loads condition assets, it uses a string identifier within the JSON to instantiate the correct concrete Condition subclass. This makes the system highly extensible, allowing new conditions to be added by simply registering them with the central codec.

A key architectural feature is the concept of **Simplicity**. The `getSimplicity` method returns an integer cost associated with evaluating the condition. The AI decision-maker can use this value to implement performance optimizations, such as evaluating low-cost conditions first to quickly prune entire branches of logic before executing expensive checks like line-of-sight calculations.

## Lifecycle & Ownership

-   **Creation:** Condition objects are **exclusively** instantiated by the Hytale AssetStore during server bootstrap or asset hot-reloading. They are deserialized from JSON definition files according to the rules defined in their respective codecs. Direct instantiation via the `new` keyword is an anti-pattern and will result in unconfigured, non-functional objects.

-   **Scope:** Once loaded, a Condition asset is a stateless, immutable definition. It persists in the `AssetStore` for the entire server session. The same instance is shared and reused by all NPCs whose AI behaviors reference it.

-   **Destruction:** The objects are garbage collected when the `AssetStore` is cleared, typically during server shutdown. The internal `WeakReference` is a mechanism used during the decoding process and does not impact the primary session-long lifecycle.

## Internal State & Concurrency

-   **State:** A Condition object's state is defined in its corresponding JSON asset and is considered **immutable** after being loaded. It contains configuration parameters (e.g., a distance threshold) but does not hold any mutable runtime state related to a specific NPC.

-   **Thread Safety:** Condition objects are inherently **thread-safe**. The primary method, `calculateUtility`, is a pure function with respect to the object's own state. It operates exclusively on world state data passed in as arguments (ArchetypeChunk, Ref, CommandBuffer). This design ensures that a single Condition instance can be safely evaluated by multiple NPC agents concurrently across different threads without locks or race conditions.

    **WARNING:** While the Condition object itself is thread-safe, the caller is responsible for ensuring that the provided world state arguments (e.g., ArchetypeChunk) are accessed in a thread-safe manner consistent with the engine's Entity Component System concurrency model.

## API Surface

The public contract of a Condition is minimal, focusing entirely on evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateUtility(...) | double | Varies | The core evaluation function. Calculates a utility score based on the provided world state. Complexity ranges from O(1) for simple stat checks to O(N) for spatial queries. |
| getSimplicity() | int | O(1) | Returns the computational cost of this condition, used for optimization by the decision-making engine. |
| getId() | String | O(1) | Returns the unique asset identifier for this condition. |
| getAssetStore() | static AssetStore | O(1) | Provides static access to the central registry for all Condition assets. |

## Integration Patterns

### Standard Usage

Conditions are not used directly. They are referenced by ID within higher-level AI assets (e.g., a Behavior or Action definition). The decision-making engine retrieves the pre-loaded instance from the AssetStore to perform an evaluation.

```java
// A hypothetical decision engine evaluating a condition
// Note: You would typically not call this directly.

// 1. Retrieve the condition from the central asset map
Condition distanceCheck = Condition.getAssetMap().get("npc_behavior_is_target_close");

if (distanceCheck != null) {
    // 2. The engine provides the full world context for evaluation
    double utilityScore = distanceCheck.calculateUtility(
        npcEntityIndex,
        worldChunk,
        targetRef,
        commandBuffer,
        evaluationContext
    );

    // 3. Use the score to influence AI behavior
    if (utilityScore > 0.8) {
        // High utility, prioritize this action
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new TargetDistanceCondition()`. This bypasses the asset loading system, leaving the object uninitialized and disconnected from its configuration. All conditions must be defined in JSON and loaded by the engine.

-   **State Modification:** Do not attempt to modify a Condition object's state after it has been loaded. They are designed to be immutable definitions. Modifying a shared asset instance at runtime will lead to unpredictable behavior across all NPCs using that condition.

-   **Ignoring Simplicity:** High-frequency evaluation of conditions with `HIGH_COST_SIMPLICITY` can create performance bottlenecks. Structure AI logic to use low-simplicity checks to guard against more expensive ones.

## Data Pipeline

The Condition class is a processing node in the NPC AI data pipeline. It transforms world state information into a quantitative utility score.

> Flow:
> JSON Asset File -> Engine AssetStore Deserialization -> In-Memory AssetMap -> AI Decision Maker -> **Condition.calculateUtility**(World State) -> Utility Score (double) -> AI Action Selection

