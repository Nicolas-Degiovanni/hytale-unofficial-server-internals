---
description: Architectural reference for TargetMemoryCountCondition
---

# TargetMemoryCountCondition

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.conditions
**Type:** Data-Driven Component

## Definition
```java
// Signature
public class TargetMemoryCountCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
The TargetMemoryCountCondition is a fundamental component within the server-side NPC Utility AI system. It functions as a "sensor" or an "input provider" for an NPC's decision-making process. Its primary role is to quantify the tactical situation by counting the number of known entities in an NPC's short-term memory.

This class extends ScaledCurveCondition, which is a critical architectural detail. It does not make a binary (true/false) decision. Instead, it provides a raw numerical input—the count of targets—which the parent ScaledCurveCondition then maps to a "utility score" via a configurable response curve. This score represents how desirable a particular action is, given the current number of targets. For example, a high number of hostile targets might produce a high utility score for an area-of-effect ability.

This component is designed to be configured entirely through data files, not instantiated directly in code. The static CODEC field is the designated factory, responsible for deserializing the condition's behavior from a definition file, typically JSON or a similar format.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the loading of NPC behavior configurations. The static CODEC field is invoked by a higher-level parser that reads an NPC's AI definition from storage. It is never instantiated with the *new* keyword in game logic.
- **Scope:** An instance of TargetMemoryCountCondition is extremely short-lived. It is typically instantiated as part of a larger evaluation context for a single NPC on a single decision-making tick. It exists only for the duration of that evaluation.
- **Destruction:** The object is eligible for garbage collection immediately after the NPC's action evaluation is complete. It holds no persistent state or external resources that require manual cleanup.

## Internal State & Concurrency
- **State:** The internal state of a TargetMemoryCountCondition instance is minimal and effectively immutable after creation. It consists of a single configuration field, targetType, which is set during deserialization by the CODEC. The class itself is stateless during execution; it does not cache results or modify its own fields when its methods are called. All necessary state is read from the TargetMemory component passed into the getInput method.
- **Thread Safety:** This class is conditionally thread-safe. The instance itself has no mutable state, so it can be safely read by multiple threads. However, the getInput method operates on an ArchetypeChunk, which is a view into the core Entity Component System (ECS) data. Thread safety is therefore dependent on the guarantees provided by the calling system, typically the server's job scheduler. The scheduler must ensure that reads from the TargetMemory component do not conflict with writes from other systems.

## API Surface
The public contract is defined by the parent ScaledCurveCondition. The primary interaction is through the overridden getInput method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | Overrides the parent method. Retrieves the TargetMemory component for the current entity and returns a count of targets based on the configured targetType. This raw count is the input for the parent's scaling curve function. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly from Java code. Its integration is declarative, defined within an NPC's behavior configuration file. A game designer specifies the condition and its parameters to influence an action's utility score.

**Conceptual Example (YAML Configuration)**
```yaml
# In an NPC's combat action definition
evaluators:
  - type: "ScaledCurve"
    # ... other curve parameters ...
    condition:
      type: "TargetMemoryCount"
      TargetType: "Hostile" # Can be Hostile, Friendly, or All
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new TargetMemoryCountCondition()`. The constructor is protected and the class is designed to be configured and created via its static CODEC. Direct instantiation bypasses the data-driven design of the AI system.
- **State Modification:** Do not attempt to modify the targetType field after the object has been created. These objects are intended to be immutable configuration holders.

## Data Pipeline
The primary function of this class is to serve as a step in the NPC action selection data pipeline. It transforms entity state into a numerical value for further processing.

> Flow:
> NPC Behavior Asset -> Codec Deserializer -> **TargetMemoryCountCondition Instance** -> `getInput()` reads TargetMemory Component -> Returns Target Count (double) -> ScaledCurveCondition maps count to Utility Score -> AI System selects highest-scoring action

