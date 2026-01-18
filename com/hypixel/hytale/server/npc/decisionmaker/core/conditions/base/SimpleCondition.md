---
description: Architectural reference for SimpleCondition
---

# SimpleCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions.base
**Type:** Transient / Abstract Base Class

## Definition
```java
// Signature
public abstract class SimpleCondition extends Condition {
```

## Architecture & Concepts
SimpleCondition is a foundational abstract class within the server-side NPC Decision Maker framework. It serves as a specialized base for all conditions that resolve to a binary, true-or-false outcome.

This class embodies the **Template Method** design pattern. It provides the final, non-overridable algorithm for utility calculation in its `calculateUtility` method, which translates a boolean result into a floating-point utility score. Subclasses are only required to implement the abstract `evaluate` method, which contains the specific logic for the condition (e.g., checking health, distance, or time of day).

The primary architectural purpose of SimpleCondition is to decouple the raw boolean logic of a condition from the utility-based scoring system used by the AI. By handling the mapping of true/false to configurable `trueValue` and `falseValue` scores, it allows behavior designers to tune AI responses in data files without altering core Java logic.

The static `ABSTRACT_CODEC` field is a critical component, indicating that all concrete implementations of SimpleCondition are designed to be deserialized from external data sources, such as NPC definition files. This makes the system highly data-driven and extensible.

## Lifecycle & Ownership
- **Creation:** Instances of SimpleCondition are not created directly using the `new` keyword. They are instantiated by the Hytale `Codec` system when parsing and loading NPC behavior configurations from data files. The `ABSTRACT_CODEC` orchestrates this deserialization process.
- **Scope:** The lifetime of a SimpleCondition instance is transient. It is typically scoped to a single decision-making evaluation cycle for a specific NPC. Once the utility scores for a set of behaviors have been calculated, the instance is no longer referenced by the core decision-making loop.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or disposal methods. Due to their short-lived nature, these objects are designed to be created and collected frequently.

## Internal State & Concurrency
- **State:** The state of a SimpleCondition instance, consisting of `trueValue` and `falseValue`, is mutable only during its creation via the `Codec` system. After deserialization, it should be treated as effectively immutable. Subclasses are strongly discouraged from introducing mutable state.
- **Thread Safety:** This class is **not thread-safe**. It is designed with the expectation that each NPC's decision-making process occurs within a single thread. Sharing a single instance of a SimpleCondition or its subclasses across multiple threads will lead to unpredictable behavior and is an unsupported pattern.

## API Surface
The public contract is inherited from the parent `Condition` class. The key interaction is through `calculateUtility`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateUtility(...) | double | O(C) | Final method that executes the condition. It invokes the internal `evaluate` method and returns `trueValue` or `falseValue` based on the result. C is the complexity of the subclass implementation. |
| getSimplicity() | int | O(1) | Returns a constant value representing the computational cost. Used by the decision-making engine to potentially prioritize simpler conditions. |
| evaluate(...) | boolean | O(C) | **Abstract method.** Subclasses must implement this to define the core boolean logic of the condition. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Instead, they create a concrete subclass and expose it to the `Codec` system. The engine then instantiates it based on data definitions.

**1. Create a concrete implementation:**
```java
// In a new file, e.g., IsNightTimeCondition.java
public class IsNightTimeCondition extends SimpleCondition {
    // The Codec for this specific implementation would be defined here.

    @Override
    protected boolean evaluate(int selfIndex, ArchetypeChunk<EntityStore> chunk, Ref<EntityStore> target, CommandBuffer<EntityStore> buffer, EvaluationContext context) {
        // Access world state via the context
        return context.getWorld().isNight();
    }
}
```

**2. Reference it in an NPC data file (conceptual JSON):**
```json
{
  "behavior": {
    "type": "IsNightTimeCondition",
    "TrueValue": 0.9,
    "FalseValue": 0.1
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new IsNightTimeCondition()`. The `Codec` system is responsible for creating and configuring these objects from data. Direct instantiation bypasses critical configuration steps.
- **Stateful Implementations:** Do not store evaluation-specific data as fields within a subclass. Conditions must be stateless to ensure they produce consistent results.
- **Bypassing Utility Calculation:** Do not call the `evaluate` method directly from outside the class. Always call `calculateUtility` to ensure the logic correctly integrates with the utility-based AI system.

## Data Pipeline
The SimpleCondition class is a processing node in the NPC behavior evaluation pipeline.

> Flow:
> NPC Definition File (JSON) -> Hytale Codec System -> **SimpleCondition Instance** -> Decision Maker Engine -> `calculateUtility()` -> Utility Score (double) -> Action Selection Logic

