---
description: Architectural reference for SelfStatPercentageCondition
---

# SelfStatPercentageCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient / Data-Driven Component

## Definition
```java
// Signature
public class SelfStatPercentageCondition extends CurveCondition {
```

## Architecture & Concepts
The SelfStatPercentageCondition is a specialized component within the server-side NPC Decision Maker framework. It serves as a data-driven logical test, allowing AI designers to create behaviors that are sensitive to an NPC's internal state.

Architecturally, this class is a leaf node in a larger AI evaluation system, likely a Utility AI system. It does not make decisions itself; instead, it provides a normalized input value (a stat percentage) to its parent, a CurveCondition. The parent class then maps this input to a "utility" score using a configurable curve. This score helps the AI system decide which behavior is most appropriate to execute at any given moment.

Its primary role is to bridge the gap between the Entity Component System (ECS), specifically the EntityStatsModule, and the abstract Decision Maker core. It translates a raw entity component state (the EntityStatMap) into a simple, normalized floating-point number that the higher-level AI logic can easily process.

The class is designed to be defined in external asset files (e.g., JSON or HOCON) and is instantiated at runtime by the engine's serialization system, using the provided static CODEC field. This pattern decouples AI logic from game code, enabling designers to tune NPC behavior without recompiling the server.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through deserialization via the static `SelfStatPercentageCondition.CODEC`. This process is typically triggered when the server loads an NPC's behavior configuration from an asset file. The `afterDecode` hook is critical, as it populates the `statIndex` field, a performance optimization that caches the integer index for the configured stat string.
- **Scope:** The object's lifetime is bound to the parent AI configuration that defines it. It is a lightweight, stateless (after initialization) object that persists as long as the NPC's behavior set is loaded in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once the owning AI configuration is unloaded or replaced. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state consists of `stat` (String) and `statIndex` (int). This state is written only once during the deserialization process. After creation, the object is effectively immutable. It does not cache any runtime data or results.
- **Thread Safety:** The core method, `getNormalisedInput`, is read-only with respect to the world state. It queries component data from an `ArchetypeChunk` but does not modify it. Assuming the underlying ECS framework guarantees thread-safe reads or proper job scheduling (i.e., not writing to the same components concurrently), this class is safe to use from multiple threads within the server's AI update loop. It contains no internal locks or synchronization primitives.

## API Surface
The primary contract is the implementation of the `getNormalisedInput` method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNormalisedInput(...) | protected double | O(1) | Retrieves the specified stat from the NPC's EntityStatMap component and returns its value as a percentage (0.0 to 1.0). This is the core logic of the condition. Asserts that the component exists on the entity. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Instead, it is configured within an NPC behavior asset file. The engine's AI system will then load, instantiate, and execute it as part of the NPC update tick.

A conceptual asset definition might look like this:

```yaml
# Example: Part of an NPC's "Flee" behavior configuration
utility: 1.0
conditions:
  - type: "SelfStatPercentageCondition"
    # The stat to monitor (e.g., "hytale:health")
    stat: "hytale:health"
    # The CurveCondition maps the stat percentage to a utility multiplier
    curve:
      type: "Linear"
      points:
        - [0.0, 1.0] # At 0% health, multiplier is 1.0 (high desire to flee)
        - [0.3, 0.0] # At 30% health, multiplier is 0.0 (no desire to flee)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SelfStatPercentageCondition()`. The object will be in an invalid state, as the `stat` and `statIndex` fields will not be initialized. This will result in a `NullPointerException` when `getNormalisedInput` is called. Always define conditions in data assets to be loaded by the engine.
- **Invalid Stat Reference:** Specifying a `stat` name in the data file that is not a valid, registered `EntityStatType` will cause the asset loader to fail with a validation error. Ensure all stat strings match a defined entity stat.

## Data Pipeline
The class functions as a specific step in the AI decision-making data flow, transforming ECS component data into a normalized value for utility calculation.

> Flow:
> NPC Behavior Asset -> Asset Deserializer -> **SelfStatPercentageCondition.CODEC** -> In-Memory `SelfStatPercentageCondition` Instance -> AI System Tick -> `getNormalisedInput` call -> Reads `EntityStatMap` Component -> Returns `double` -> `CurveCondition` Parent -> Final Utility Score

