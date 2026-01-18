---
description: Architectural reference for RegenHealthCondition
---

# RegenHealthCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Object

## Definition
```java
// Signature
public class RegenHealthCondition extends Condition {
```

## Architecture & Concepts
The RegenHealthCondition is a concrete implementation of the `Condition` abstraction, designed to be used within the server-side Entity Stats module. Its primary role is to act as a predicate in a data-driven evaluation chain, determining if a specific game mechanic, such as a status effect or stat modifier, should be applied to an entity.

This class is not intended for direct use in procedural game logic. Instead, it is designed to be instantiated by the asset loading system from game data files (e.g., JSON). The static `CODEC` field is the key integration point for this deserialization process, allowing game designers to specify this condition by name within an asset definition.

**WARNING:** The current implementation of the core evaluation method, `eval0`, is a stub. It unconditionally returns `true`, regardless of the entity's state or the current time. This means that, in its present form, this condition always passes and serves as a placeholder or a pass-through component in any evaluation logic.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the asset deserialization pipeline, which uses the public static `CODEC` field. It is not intended to be instantiated manually with the `new` keyword.
- **Scope:** Extremely short-lived and transient. An instance is typically created on-demand during the evaluation of an entity's stats and is discarded immediately after the evaluation is complete. It holds no long-term state.
- **Destruction:** Managed by the Java Garbage Collector. As soon as the evaluation logic that created it completes, the instance becomes unreachable and is eligible for collection.

## Internal State & Concurrency
- **State:** Immutable. The only state is the `inverse` boolean inherited from its `Condition` parent, which is set at construction time and cannot be changed. The class itself contains no mutable fields.
- **Thread Safety:** This class is inherently thread-safe. Its immutability and the lack of side effects in its evaluation method ensure that a single instance can be safely evaluated by multiple threads concurrently without risk of data corruption or race conditions.

## API Surface
The primary contract is fulfilled by implementing the `eval0` method from the parent `Condition` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Evaluates the condition. **WARNING:** This is a stub implementation that always returns `true`. It does not perform any checks related to health regeneration. |

## Integration Patterns

### Standard Usage
This component is not used directly in Java code. It is declared within data assets, which are then processed by the server's entity stat evaluation engine.

```json
// Example: Declaring the condition within a game asset file
{
  "id": "example.effect.fast_regen",
  "modifiers": [
    {
      "stat": "hytale:health_regen_rate",
      "operation": "multiply",
      "value": 2.0
    }
  ],
  "conditions": [
    {
      "type": "RegenHealthCondition",
      "inverse": false
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new RegenHealthCondition()` in game logic. The system is designed to be data-driven. Modifying behavior should be done by changing the asset files, not by writing procedural code that creates conditions.
- **Assuming Functionality:** Do not use this condition with the expectation that it will check an entity's ability to regenerate health. Its current implementation is a placeholder and will not function as its name implies.

## Data Pipeline
The flow of data for this component is from asset definition to a boolean evaluation result.

> Flow:
> Game Asset File (JSON) -> Asset Deserializer (using `CODEC`) -> **RegenHealthCondition Instance** -> Entity Stat Evaluator -> Boolean Result (`true`)

