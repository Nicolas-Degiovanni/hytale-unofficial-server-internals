---
description: Architectural reference for TimeOfDayCondition
---

# TimeOfDayCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient

## Definition
```java
// Signature
public class TimeOfDayCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
The TimeOfDayCondition is a specialized component within the server-side NPC Decision Maker framework. This framework operates on principles of Utility AI, where various environmental and stateful factors are evaluated to produce a "utility score" for potential actions. The action with the highest score is then executed by the NPC.

This class serves as a sensor, providing a single, normalized input to the utility calculation: the current in-game time of day. It does not compute the final utility score itself. Instead, it inherits from ScaledCurveCondition, which is responsible for taking the raw time value and mapping it to a score using a pre-configured evaluation curve.

Architecturally, it acts as a bridge between a global engine state, the WorldTimeResource, and the specific decision-making context of an individual NPC. Its sole responsibility is to query the world time and present it to the parent evaluation system. For example, a nocturnal creature's behavior file might pair this condition with a curve that produces high utility scores during nighttime hours (e.g., 18.0 to 6.0).

## Lifecycle & Ownership
- **Creation:** TimeOfDayCondition instances are not created directly using the new keyword. They are instantiated by the engine's serialization system via the provided static CODEC field. This typically occurs when the server loads an NPC's behavior profile from a data file (e.g., a JSON asset).
- **Scope:** The object is stateless and effectively immutable. A single instance, as part of a loaded behavior profile, can be shared and reused by thousands of NPCs of the same type. Its lifetime is tied to the lifetime of the loaded behavior profile asset.
- **Destruction:** The object is garbage collected when its containing behavior profile is unloaded from memory, for instance, when a world or zone is shut down.

## Internal State & Concurrency
- **State:** Immutable. This class contains no internal member fields and does not cache any data. Each invocation of its logic fetches the time directly from the authoritative WorldTimeResource.
- **Thread Safety:** This class is inherently thread-safe. As a stateless object, it can be evaluated by multiple NPC agents on different worker threads without risk of data corruption. The responsibility for thread-safe access to the underlying WorldTimeResource lies with the engine's resource manager, not this class.

## API Surface
The primary contract is fulfilled by overriding a protected method from its parent. The Decision Maker framework interacts with the public API of the parent ScaledCurveCondition, not this class directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | protected double | O(1) | Overrides the parent method to provide the input for the utility curve. It fetches the current time from the global WorldTimeResource and returns it as a value from 0.0 to 24.0. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly in procedural Java code. It is designed to be declared within data-driven NPC behavior assets. The engine's Decision Maker system will then instantiate and integrate it automatically.

**WARNING:** The following example is a conceptual representation in YAML. The actual in-game format may differ.

```yaml
# Example: Part of a hypothetical NPC behavior asset
scorers:
  - name: "IsNightTime"
    weight: 1.5
    condition:
      # The engine maps this 'type' to TimeOfDayCondition.CODEC
      type: "TimeOfDayCondition"
      # The parent ScaledCurveCondition uses this curve to evaluate the input
      curve:
        type: "StepCurve"
        points:
          - { input: 0.0, output: 1.0 }   # High score from midnight...
          - { input: 6.0, output: 0.0 }   # ...to 6 AM
          - { input: 18.0, output: 0.0 }  # Low score from 6 AM...
          - { input: 18.1, output: 1.0 }  # ...until 6 PM
          - { input: 24.0, output: 1.0 }  # High score until midnight
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TimeOfDayCondition()`. The constructor is protected and creating an instance this way bypasses the necessary configuration (like the evaluation curve) handled by the parent class and the serialization framework.
- **Manual Evaluation:** Do not attempt to acquire an instance and call its methods manually. It is designed to be managed and executed exclusively by the NPC Decision Maker system, which provides the necessary evaluation context.

## Data Pipeline
The flow of data for this component is linear and unidirectional. It acts as an initial sensor in a larger utility calculation chain.

> Flow:
> WorldTimeResource (Global State) -> **TimeOfDayCondition.getInput()** -> ScaledCurveCondition (Curve Mapping) -> Utility Score -> Decision Maker (Action Selection)

