---
description: Architectural reference for TotalSustainedDamageCondition
---

# TotalSustainedDamageCondition

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.conditions
**Type:** Component Logic / Data-Driven Rule

## Definition
```java
// Signature
public class TotalSustainedDamageCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
The TotalSustainedDamageCondition is a specific implementation within the server-side NPC Utility AI framework. It serves as a "Consideration" that allows an NPC to factor its own survivability into its decision-making process. Its primary function is to translate the raw, cumulative damage an NPC has taken during a combat encounter into a normalized utility score.

Architecturally, this class acts as a bridge between the Entity-Component-System (ECS) data layer and the abstract AI decision-making engine. It queries a specific component, DamageMemory, for state and feeds that state into its parent, ScaledCurveCondition, which then uses a data-defined curve to calculate the final score.

This component is designed to be entirely data-driven. Game designers define its behavior and tune its response curve in external asset files, which are deserialized at runtime using the provided BuilderCodec. This decouples high-level AI logic from low-level engine code.

### Lifecycle & Ownership
- **Creation:** Instances are created by the engine's Codec system when an NPC's AI behavior asset is loaded from disk. It is not intended for manual instantiation.
- **Scope:** The object's lifetime is bound to the loaded NPC behavior asset. It is effectively immutable after creation and is reused across all NPCs that share this specific AI configuration.
- **Destruction:** The object is eligible for garbage collection when the corresponding AI asset is unloaded by the server, for instance, during a zone change or server shutdown.

## Internal State & Concurrency
- **State:** This class is stateless. It holds no mutable fields and does not cache any data between calls. All state it operates on is read directly from the DamageMemory component associated with the NPC entity being evaluated. The only internal data is the immutable response curve configuration inherited from ScaledCurveCondition, which is set upon creation.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread responsible for NPC updates. The ECS structures it interacts with, such as ArchetypeChunk, are designed for single-threaded access to guarantee data integrity and avoid race conditions.

## API Surface
The primary API is the protected contract with its parent class. Direct invocation is not a standard use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setupNPC(Holder) | void | O(1) | Initialization hook. Ensures the target NPC entity has a DamageMemory component. Throws an error if the component cannot be added. |
| getInput(...) | double | O(1) | *Protected.* Retrieves the total damage from the NPC's DamageMemory component. This value is the input for the parent's curve evaluation. |

## Integration Patterns

### Standard Usage
This condition is not used directly in Java code. Instead, it is specified within an NPC's AI definition file. A game designer would reference it by its registered name and configure the parent ScaledCurveCondition parameters.

```yaml
# Example: NPC AI Behavior Asset (npc_behavior.yml)
decisionMaker:
  evaluators:
    - name: "RetreatConsideration"
      action: "Flee"
      conditions:
        - type: "TotalSustainedDamageCondition"
          # Parameters for the parent ScaledCurveCondition
          curve: "LINEAR"
          inputMin: 0
          inputMax: 500 # At 500 total damage, utility score will be 1.0
          outputMin: 0.1
          outputMax: 1.0
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TotalSustainedDamageCondition()`. The object is useless without the curve configuration that is injected by the codec system during asset loading.
- **Missing Component Dependency:** Using this condition on an NPC entity that does not have the DamageMemory component will result in a runtime error during the `setupNPC` phase. All entities using this logic must have their state tracked by DamageMemory.

## Data Pipeline
This class operates as a single step in the larger NPC AI evaluation pipeline. It transforms raw component state into a normalized score.

> Flow:
> Damage Event → DamageMemory Component Update → NPC AI Tick → **TotalSustainedDamageCondition.getInput()** → ScaledCurveCondition Evaluation → Utility Score → Action Selection Logic

