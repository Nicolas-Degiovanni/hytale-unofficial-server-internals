---
description: Architectural reference for TargetStatPercentageCondition
---

# TargetStatPercentageCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient Data Object

## Definition
```java
// Signature
public class TargetStatPercentageCondition extends CurveCondition {
```

## Architecture & Concepts
The TargetStatPercentageCondition is a specialized component within the server-side NPC Decision Maker framework. It functions as a data-driven "sensor" that allows an AI to evaluate the state of a target entity.

Architecturally, this class is a leaf node in a larger Utility AI or Behavior Tree system. It is not a standalone service but a configurable piece of logic. Its primary role is to bridge the abstract Decision Maker system with the concrete Entity Component System (ECS). Specifically, it queries the EntityStatsModule to retrieve a specific stat (e.g., health, mana, stamina) from a target entity.

By extending CurveCondition, it transforms the raw stat percentage into a normalized input value (between 0.0 and 1.0). This value is then fed into a utility curve, which produces a final "utility score". This score helps the AI decide which action to take. For example, an NPC might have a high utility score for "Flee" when this condition reports that its own health percentage is low.

The class is designed to be defined in external asset files (like JSON) and loaded at runtime, enabling designers to tweak AI behavior without changing game code. The static CODEC field is the key to this data-driven architecture, handling deserialization, validation, and post-processing.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the constructor. They are instantiated by the Hytale Codec system during server startup or when NPC behavior assets are loaded. The static CODEC field defines the deserialization and initialization logic, including the critical step of caching the stat index.
- **Scope:** The lifetime of a TargetStatPercentageCondition instance is tied to the lifecycle of the parent AI behavior asset it is part of. It persists in memory as long as that AI definition is loaded.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection when the AI behavior definition is unloaded and no longer referenced.

## Internal State & Concurrency
- **State:** The internal state, consisting of the stat name and its cached index, is **effectively immutable** after initialization. The state is populated once during the `afterDecode` phase of the codec lifecycle and is not modified thereafter. This design ensures predictable behavior during evaluation.
- **Thread Safety:** This class is **not thread-safe** and contains no internal synchronization mechanisms. It is designed to be executed exclusively within the single-threaded context of the NPC Decision Maker's evaluation loop for a given world or zone. Accessing an instance from multiple threads will lead to unpredictable behavior and race conditions when interacting with the CommandBuffer and ECS.

## API Surface
The primary interaction is through the `evaluate` method inherited from its parent. The core logic resides in the protected `getNormalisedInput` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNormalisedInput(...) | double | O(1) | **(Protected)** Fetches the specified stat from the target entity's EntityStatMap component and returns its value as a percentage. Returns Double.MAX_VALUE if the target is null or invalid. |

## Integration Patterns

### Standard Usage
A game developer or AI designer does not interact with this class directly in Java code. Instead, they define its properties within an NPC behavior asset file. The system then uses the instance during the AI evaluation tick. The following conceptual example illustrates how the Decision Maker system might use a loaded condition.

```java
// This code is conceptual and executed by the Decision Maker engine.
// 'condition' is an instance of TargetStatPercentageCondition loaded from an asset.

// Get the current evaluation context for an NPC
EvaluationContext context = npc.getEvaluationContext();

// The engine retrieves the target from the context
Ref<EntityStore> target = context.getTarget();

// The engine invokes the evaluation, which internally calls getNormalisedInput
// The CommandBuffer is provided by the engine's current update tick.
double statPercentage = condition.getNormalisedInput(
    npc.getEntityIndex(),
    npc.getArchetypeChunk(),
    target,
    engine.getCommandBuffer(),
    context
);

// The result is then used to calculate a utility score.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new TargetStatPercentageCondition()`. This bypasses the critical `CODEC` logic, leaving the internal `stat` and `statIndex` fields uninitialized. This will result in NullPointerExceptions or incorrect behavior during evaluation.
- **State Mutation:** Do not attempt to use reflection or other means to modify the `stat` or `statIndex` fields after initialization. The object is designed to be immutable post-creation.
- **Cross-Thread Access:** Do not share instances of this class or its evaluation context across different threads. All interactions must be confined to the owning AI system's update thread.

## Data Pipeline
The primary function of this class is to act as a step in the data transformation pipeline that converts a high-level AI goal into a concrete action.

> Flow:
> NPC Behavior Asset (JSON) -> Hytale Codec System -> **TargetStatPercentageCondition Instance** -> Decision Maker Evaluation -> ECS Query via CommandBuffer -> Stat Percentage (double) -> Utility Curve -> Final Utility Score

