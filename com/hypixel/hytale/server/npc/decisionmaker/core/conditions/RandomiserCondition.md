---
description: Architectural reference for RandomiserCondition
---

# RandomiserCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient

## Definition
```java
// Signature
public class RandomiserCondition extends Condition {
```

## Architecture & Concepts
The RandomiserCondition is a fundamental component within the server-side NPC Utility AI system. It functions as a leaf node in a behavior evaluation tree, designed to introduce non-determinism into an NPC's decision-making process.

Its primary role is not to evaluate world state, but to provide a "jitter" value. In a Utility AI, actions are scored based on various conditions. When two or more actions have nearly identical utility scores, the NPC's behavior can become predictable and robotic. The RandomiserCondition breaks these ties by adding a small, bounded random value to the final utility calculation. This ensures that an NPC might choose a slightly suboptimal but different action, leading to more organic and less repetitive behavior.

It is configured declaratively in NPC behavior assets and is invoked by the core Decision Maker engine during its evaluation cycle for a specific entity.

### Lifecycle & Ownership
-   **Creation:** This object is not intended for manual instantiation. It is exclusively created by the Hytale codec system during the loading of NPC behavior assets. The static CODEC field defines the deserialization logic from a data format like JSON or HOCON.
-   **Scope:** The lifecycle of a RandomiserCondition instance is bound to the lifecycle of the NPC behavior asset that defines it. It is loaded into memory once and can be shared by all NPC entities that use that specific behavior configuration.
-   **Destruction:** The object is eligible for garbage collection when its parent behavior asset is unloaded, typically during a server shutdown, world change, or a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** The internal state consists of two double-precision floating-point numbers: *minValue* and *maxValue*. These values are set once during deserialization and are not modified thereafter. The object is therefore effectively **immutable** after its initial construction by the codec system.
-   **Thread Safety:** This class is **thread-safe**. The evaluation method, calculateUtility, retrieves a random number generator via ThreadLocalRandom. This practice avoids lock contention on a shared Random instance, making it highly performant and safe for use in the server's multi-threaded entity processing environment. All internal state is read-only during execution.

## API Surface
The public contract is minimal, focusing entirely on its integration with the decision-making engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateUtility(...) | double | O(1) | Returns a random double between the configured minValue and maxValue. This is the core function invoked by the AI engine. |
| getSimplicity() | int | O(1) | Returns a constant value representing the low computational cost of this condition. Used by the Decision Maker for performance heuristics. |

## Integration Patterns

### Standard Usage
This component is not used imperatively in code. It is defined declaratively within an NPC behavior asset file. The engine then loads and executes it automatically.

A hypothetical asset definition might look like this:

```yaml
# Example: Part of an NPC's behavior definition asset
considerations:
  - type: "RandomiserCondition"
    # Adds a random jitter between 0.0 and 0.15 to the final score
    MinValue: 0.0
    MaxValue: 0.15
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new RandomiserCondition()`. This bypasses the codec system, leaving the *minValue* and *maxValue* fields uninitialized (defaulting to 0.0), which nullifies the component's purpose.
-   **State Mutation:** Do not use reflection or other means to modify the *minValue* or *maxValue* fields after the object has been loaded. Behavior assets are often shared, and mutating this state will cause unpredictable side effects for all NPCs using the asset.

## Data Pipeline
The RandomiserCondition acts as a value generator within the broader NPC decision evaluation pipeline.

> Flow:
> NPC Behavior Asset File -> Server Asset Loader (using CODEC) -> In-Memory **RandomiserCondition** Instance -> Decision Maker Evaluation -> Utility Score Contribution -> Final Action Selection

