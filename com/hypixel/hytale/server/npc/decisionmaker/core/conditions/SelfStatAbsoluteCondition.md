---
description: Architectural reference for SelfStatAbsoluteCondition
---

# SelfStatAbsoluteCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient

## Definition
```java
// Signature
public class SelfStatAbsoluteCondition extends ScaledCurveCondition {
```

## Architecture & Concepts

The SelfStatAbsoluteCondition is a key component within the server-side NPC Utility AI system. It functions as a leaf node in a decision-making structure, responsible for providing a raw numerical input based on an NPC's own statistics.

This class does not return a simple boolean; instead, it provides a `double` value representing the absolute magnitude of a specific entity statistic (e.g., Health, Mana, Rage). This raw value is then fed into its parent class, ScaledCurveCondition, which transforms the input into a final "utility score" by mapping it against a configurable curve. This architecture allows game designers to create nuanced AI behaviors where an NPC's desire to perform an action changes dynamically with its stats. For example, the desire to use a healing ability might increase as the NPC's health stat decreases.

Crucially, SelfStatAbsoluteCondition is designed to be data-driven. Its behavior, specifically which statistic it monitors, is configured via the static CODEC field. This allows designers to define and tune NPC logic in external asset files without modifying engine source code. The class operates directly on the Entity Component System (ECS) data, reading from the EntityStatMap component associated with the NPC being evaluated.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the constructor. They are deserialized from configuration assets by the Hytale Codec system, using the public static CODEC field. The `afterDecode` hook is critical for performance, as it caches the integer index of the target stat, avoiding repeated string lookups during evaluation.
-   **Scope:** An instance's lifetime is bound to the NPC behavior asset that defines it. It is effectively immutable after being loaded and persists as long as the parent behavior configuration is in memory.
-   **Destruction:** The object is eligible for garbage collection when the corresponding NPC behavior asset is unloaded from the server.

## Internal State & Concurrency
-   **State:** The internal state consists of `stat` (a String) and `statIndex` (an int). This state is set once during deserialization and is treated as immutable for the object's lifetime. The class itself holds no per-evaluation state; all dynamic data is read from the ECS via the provided EvaluationContext.
-   **Thread Safety:** This class is inherently thread-safe. Its internal state is immutable post-creation, and the `getInput` method is a pure function with respect to the class instance. However, the caller is responsible for ensuring that the provided `ArchetypeChunk` and `EvaluationContext` are not accessed concurrently by multiple threads in a way that violates the ECS's threading model. It is designed to be executed within a single-threaded context for a given NPC's decision-making tick.

## API Surface

The primary interaction point is the `getInput` method, which is an override of a protected method from its parent. The public contract is fulfilled by the evaluation logic in the parent `ScaledCurveCondition`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(...) | double | O(1) | Retrieves the absolute value of a configured stat from the NPC's EntityStatMap component. Asserts that the component exists. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use in procedural Java code. It is designed to be defined declaratively within an NPC behavior asset file. The engine's decision-making module is responsible for its instantiation and execution.

A designer would configure it in a JSON or similar asset like this (conceptual example):

```json
{
  "type": "SelfStatAbsoluteCondition",
  "stat": "hytale:health",
  "curve": { ... }
}
```

The system then uses the deserialized object during the NPC's update tick.

```java
// Engine-level code (conceptual)
// 'condition' is an instance of SelfStatAbsoluteCondition loaded from an asset
double utilityScore = condition.evaluate(self, target, context);

// The engine then uses this score to help select an action
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SelfStatAbsoluteCondition()`. The default constructor exists for the codec system. Manual instantiation will result in a non-functional object with a null `stat` and an invalid `statIndex`, leading to NullPointerExceptions at runtime.
-   **State Mutation:** Do not attempt to modify the `stat` or `statIndex` fields after the object has been created by the codec. This would break the immutability guarantee and could lead to unpredictable behavior.

## Data Pipeline

The flow of data for this component is unidirectional, from configuration and the live game state to a final utility score.

> Flow:
> NPC Behavior Asset (JSON) -> **CODEC Deserializer** -> In-memory **SelfStatAbsoluteCondition** instance -> Game Tick triggers AI evaluation -> `getInput` reads `EntityStatMap` from ECS -> Raw stat value is returned -> `ScaledCurveCondition` (parent) processes value -> Final Utility Score

