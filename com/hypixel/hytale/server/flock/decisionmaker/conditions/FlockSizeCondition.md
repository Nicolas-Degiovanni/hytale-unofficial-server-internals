---
description: Architectural reference for FlockSizeCondition
---

# FlockSizeCondition

**Package:** com.hypixel.hytale.server.flock.decisionmaker.conditions
**Type:** Transient Component

## Definition
```java
// Signature
public class FlockSizeCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
The FlockSizeCondition is a specialized component within the server-side NPC Decision Making framework. It functions as a leaf node in a behavior tree or a data provider for a utility-based AI scorer. Its sole responsibility is to measure the current size of an NPC's associated flock and provide this count as a raw input value.

This class operates entirely within the server's Entity Component System (ECS). It does not perform calculations itself but rather queries the state of ECS components, specifically the FlockMembership of the evaluating entity and the EntityGroup of the referenced flock.

The raw flock size returned by this condition is consumed by its parent class, ScaledCurveCondition. The parent class then maps this raw input value (e.g., a flock size of 5) to a normalized score (e.g., 0.75) using a configurable evaluation curve. This final score is what the broader AI system uses to decide which behavior to execute.

## Lifecycle & Ownership
- **Creation:** FlockSizeCondition instances are not created directly via code. They are instantiated by the server's asset loading pipeline using the provided static CODEC. This occurs when the server loads NPC behavior profiles from configuration files (e.g., JSON assets).
- **Scope:** An instance of FlockSizeCondition is effectively a stateless, immutable configuration object. A single instance may be shared across all NPC entities that use the same behavior profile. Its lifetime is tied to the loaded asset, not to any individual NPC.
- **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding NPC behavior asset, for example during a zone change or server shutdown.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the parameters passed during method invocation. All state is read from the ECS via the provided ArchetypeChunk and CommandBuffer.
- **Thread Safety:** This class is **unconditionally thread-safe**. Its stateless nature ensures that it can be safely evaluated by multiple AI update threads concurrently without risk of race conditions or data corruption. Thread safety of the underlying ECS data is managed by the CommandBuffer and the calling scheduler.

## API Surface
The public contract is fulfilled by the `evaluate` method inherited from its parent. The core logic resides in the protected `getInput` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | Retrieves the size of the NPC's current flock. Returns 1.0 if the entity is not in a flock or the flock reference is invalid. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly from Java code. Instead, it is configured within an NPC's behavior asset file. The decision-making system invokes it automatically during the AI evaluation tick.

**Warning:** The following example is a conceptual representation. The actual schema may differ.

```json
// Example: Part of an NPC behavior asset
{
  "id": "behavior.wolf.pack_howl",
  "conditions": [
    {
      "type": "FlockSizeCondition",
      "curve": "Linear",
      "minInput": 2,
      "maxInput": 10
    }
  ],
  "actions": [ ... ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new FlockSizeCondition()`. The object will lack the necessary curve configuration provided by the asset loader and is meaningless outside the decision-making framework.
- **Misinterpreting Default Value:** Do not rely on this condition to check *for the existence* of a flock. An entity with no FlockMembership component will return a value of 1.0, which may pass certain curve thresholds. Use a dedicated `HasComponentCondition` for existence checks.
- **External Invocation:** Do not call methods on this class from outside the server's NPC decision-making system. The context parameters (ArchetypeChunk, CommandBuffer) are highly specific to the AI update loop and cannot be easily fabricated.

## Data Pipeline
The flow of data for this condition begins with the AI scheduler and ends with a normalized score being used for behavior selection.

> Flow:
> AI Update Tick -> Evaluates NPC Behavior -> **FlockSizeCondition.getInput()** -> Reads FlockMembership Component -> Reads EntityGroup Component Size -> Returns Raw Count -> ScaledCurveCondition Applies Curve -> Returns Final Score -> AI System Selects Action

