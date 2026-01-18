---
description: Architectural reference for TimeSinceLastUsedCondition
---

# TimeSinceLastUsedCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient / Strategy

## Definition
```java
// Signature
public class TimeSinceLastUsedCondition extends ScaledCurveCondition {
```

## Architecture & Concepts
TimeSinceLastUsedCondition is a component within the server-side NPC Utility AI framework. It functions as a specific *Condition* used to evaluate the desirability of an AI behavior, known as an *Option*.

Its sole responsibility is to measure the real-world time elapsed since the associated AI Option was last executed. This raw time value, measured in seconds, serves as the input for its parent class, ScaledCurveCondition. The parent class then uses a pre-configured evaluation curve to transform this duration into a final utility score.

This architecture enables a powerful and data-driven approach to implementing time-based AI logic. Designers can create sophisticated cooldowns, or behaviors that increase in priority the longer they remain unused, without writing any code. For example, an NPC's healing ability can be configured with this condition to make it a more attractive choice as more time passes since its last use.

The class is designed to be instantiated from game data files via its static CODEC, making it a modular and reusable building block for defining complex NPC behaviors.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the engine's BuilderCodec system during the loading of NPC behavior configurations from server assets. It is never created manually in code.
- **Scope:** The object is effectively a stateless template. Its lifetime is bound to the NPC behavior definition it is a part of. A single instance is often shared across all NPC agents that utilize the same behavior configuration.
- **Destruction:** The object is garbage collected when its parent NPC behavior definition is unloaded from memory, such as during a world or zone change.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It holds no internal fields and its behavior is entirely determined by the arguments passed to its methods, specifically the EvaluationContext.
- **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature allows it to be called concurrently from multiple threads—a common scenario in the NPC decision-making loop—without any risk of race conditions or data corruption. The responsibility for managing the state it reads (from EvaluationContext) lies with the calling AI system.

## API Surface
The public contract is inherited from its parent. The primary implementation detail is the `getInput` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | **(Protected)** Calculates the elapsed time in seconds since the associated AI Option was last used. The timestamp is retrieved from the EvaluationContext. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, programmatic use. It is designed to be declared within data files (e.g., JSON or a similar format) that define an NPC's decision-making logic. The engine's AI system automatically instantiates and invokes it as part of the behavior evaluation pipeline.

A conceptual data definition might look like this:
```yaml
# This is a conceptual example of how this class would be defined in data.
# It is NOT executable Java code.

option: "UseSpecialAttack"
considerations: [
  {
    # The engine maps this type name to TimeSinceLastUsedCondition.CODEC
    type: "TimeSinceLastUsedCondition"

    # Parameters for the parent ScaledCurveCondition
    curve: "LinearGrowth"
    inputMin: 0.0      # 0 seconds since last use
    inputMax: 30.0     # 30 seconds since last use
    outputMin: 0.0     # Utility score of 0
    outputMax: 1.0     # Utility score of 1
  }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TimeSinceLastUsedCondition()`. The protected constructor and CODEC pattern enforce that this class must be managed by the engine's data-driven systems. Direct instantiation bypasses this and will fail in a production environment.
- **Stateful Subclassing:** Avoid extending this class to add internal state. Its statelessness is a critical design feature for ensuring thread safety and reusability in a highly concurrent AI system.

## Data Pipeline
This component acts as a data transformer within the NPC decision evaluation pipeline, converting a timestamp from the context into a raw time duration.

> Flow:
> NPC Decision Maker Loop → Evaluates AI Option → **TimeSinceLastUsedCondition.getInput()** → ScaledCurveCondition.evaluate() → Final Utility Score → Action Selection System

