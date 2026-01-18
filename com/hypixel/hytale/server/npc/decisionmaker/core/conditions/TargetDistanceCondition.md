---
description: Architectural reference for TargetDistanceCondition
---

# TargetDistanceCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient Component

## Definition
```java
// Signature
public class TargetDistanceCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
The TargetDistanceCondition is a fundamental component within the server-side NPC Utility AI system. It does not make a decision on its own; rather, it serves as a raw data provider for higher-level evaluation logic. Its sole responsibility is to calculate the Euclidean distance between an NPC and its current target entity.

This class is a concrete implementation of the ScaledCurveCondition. The value it computes—the distance—is fed directly into the evaluation curve defined in its parent. The parent class then maps this raw distance onto a normalized utility score (typically 0.0 to 1.0). This final score is what the AI Decision Maker uses to weigh different potential behaviors.

For example, a "Melee Attack" behavior might use a TargetDistanceCondition with an inverse curve, producing a high utility score at close distances. Conversely, a "Shoot Arrow" behavior might use a curve that peaks at a medium range. This component is therefore a critical building block for creating spatially-aware AI behaviors.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via constructor in game logic. They are deserialized from entity or AI behavior asset files (e.g., JSON) by the Hytale **BuilderCodec** system. The static CODEC field acts as the factory for this process.
- **Scope:** The lifetime of a TargetDistanceCondition instance is bound to the AI behavior definition it is part of. As it is stateless, a single instance can be shared and reused by all NPCs that use the same behavior configuration.
- **Destruction:** The object is garbage collected when its parent AI behavior definition is unloaded from memory. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** **Stateless and Immutable.** After being deserialized and configured, an instance of TargetDistanceCondition does not change. All calculations are performed using data passed in via method arguments from the Entity Component System (ECS).
- **Thread Safety:** **Inherently Thread-Safe.** The class contains no mutable fields and its core logic in the getInput method is a pure function of its inputs. It can be safely evaluated by multiple AI agents in parallel, provided that the ECS context (ArchetypeChunk, CommandBuffer) passed to it is handled correctly by the calling scheduler.

## API Surface
The primary contract is the protected method getInput, which is invoked by the parent ScaledCurveCondition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | Calculates the distance to the target. Returns Double.MAX_VALUE if the target is null or invalid. |

**WARNING:** The return value is a raw distance, not a normalized utility score. Consuming this value directly without processing it through an evaluation curve is a design error.

## Integration Patterns

### Standard Usage
This component is not used directly in Java code by developers. Instead, it is declared within an AI behavior data file. The engine's codec system instantiates and integrates it into the decision-making system.

A conceptual data definition might look like this:
```json
// Example AI Behavior Snippet
{
  "type": "TargetDistanceCondition",
  "curve": {
    "type": "inverse",
    "max_value": 10.0
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TargetDistanceCondition()`. The object would be useless, as it would lack the critical evaluation curve data that is configured on its parent during deserialization.
- **Invalid Target Handling:** Do not assume the target is always valid. This condition correctly returns `Double.MAX_VALUE` for invalid targets. Higher-level logic must be prepared to handle this sentinel value, which typically results in a zero utility score after curve evaluation.

## Data Pipeline
The flow of data through this component is linear and part of a larger AI evaluation tick.

> Flow:
> AI Behavior Evaluation -> **TargetDistanceCondition.getInput()** -> [Raw Distance] -> ScaledCurveCondition.evaluate() -> [Utility Score] -> Decision Maker System<ctrl63>

