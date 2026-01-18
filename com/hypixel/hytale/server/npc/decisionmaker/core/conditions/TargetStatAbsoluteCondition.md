---
description: Architectural reference for TargetStatAbsoluteCondition
---

# TargetStatAbsoluteCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Configuration Component

## Definition
```java
// Signature
public class TargetStatAbsoluteCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
TargetStatAbsoluteCondition is a specific implementation within the server-side NPC Utility AI framework. It serves as a leaf node in a behavior tree or decision-making graph, providing a quantitative input based on the current state of a target entity.

This class is not a standalone service but rather a data-driven component that represents a single logical test: "What is the current value of a specific stat on my target?". It acts as a bridge between the Entity Component System (ECS), specifically the EntityStatsModule, and the AI's decision-making core.

Its direct parent, ScaledCurveCondition, consumes the raw stat value returned by this class and maps it to a normalized utility score using a predefined curve. This allows game designers to model complex behaviors, such as an NPC's desire to attack a target increasing exponentially as the target's health decreases.

The class leverages a common engine optimization pattern: a string-based identifier, *stat*, is resolved to a faster integer index, *statIndex*, during the asset decoding phase. This avoids costly string comparisons or hash map lookups during the high-frequency AI evaluation loop.

## Lifecycle & Ownership
- **Creation:** Instances are not created programmatically using the *new* keyword. They are deserialized from AI behavior asset files by the engine's asset loading system via the static CODEC field. This process is part of the server's startup or world loading sequence.
- **Scope:** The object's lifetime is bound to the AI behavior asset that defines it. It is effectively an immutable configuration object that persists as long as the corresponding NPC behavior is loaded in memory.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated AI asset bundle, typically during a server shutdown or a dynamic content reload.

## Internal State & Concurrency
- **State:** The internal state, consisting of *stat* and *statIndex*, is mutable only during the initial deserialization process managed by its CODEC. The *afterDecode* hook populates *statIndex*, after which the object's state should be considered immutable for the remainder of its lifecycle. It holds no runtime-generated or cached data.
- **Thread Safety:** This class is conditionally thread-safe. The getInput method contains no internal synchronization and does not modify its own state. Its safety is therefore entirely dependent on the thread-safety guarantees of the CommandBuffer and the underlying ECS components it accesses. It is designed to be called from the main server thread or a dedicated, single-threaded AI evaluation context.

**WARNING:** Calling getInput from multiple threads without external synchronization on the CommandBuffer or EntityStore will lead to race conditions and undefined behavior.

## API Surface
The public contract is minimal, primarily exposing the logic through the overridden getInput method from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | Retrieves the absolute value of the configured stat from the target entity. Returns Double.MAX_VALUE if the target is null or invalid. |

## Integration Patterns

### Standard Usage
This condition is not intended to be used directly from Java code. It is designed to be defined declaratively within an NPC behavior asset file (e.g., a JSON file). The engine's DecisionMaker system will then automatically instantiate and evaluate it as part of an NPC's update tick.

*Example AI Behavior Snippet (Conceptual JSON)*
```json
{
  "id": "evaluate_target_health",
  "type": "TargetStatAbsoluteCondition",
  "stat": "hytale:health",
  "curve": "linear_inverse"
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TargetStatAbsoluteCondition()`. This will create a non-functional object, as the critical *stat* and *statIndex* fields will not be initialized. All instances must be created via the engine's asset loading and codec system.
- **State Mutation:** Do not modify the public *stat* or *statIndex* fields after the object has been initialized by the codec. Doing so will break the contract with the AI system and lead to unpredictable behavior or exceptions.

## Data Pipeline
The primary function of this class is to act as a data source for the broader AI evaluation pipeline. It queries the game state and transforms a specific data point into a raw input for further processing.

> Flow:
> AI Behavior Asset File -> Engine Asset Loader -> **TargetStatAbsoluteCondition.CODEC** -> In-Memory Object -> AI Evaluation System calls `getInput` -> CommandBuffer queries ECS for `EntityStatMap` -> Raw stat value is returned -> `ScaledCurveCondition` parent class processes value -> Final Utility Score

