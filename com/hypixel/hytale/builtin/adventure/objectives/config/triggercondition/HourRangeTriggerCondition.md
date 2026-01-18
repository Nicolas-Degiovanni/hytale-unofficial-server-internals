---
description: Architectural reference for HourRangeTriggerCondition
---

# HourRangeTriggerCondition

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.triggercondition
**Type:** Configuration Object

## Definition
```java
// Signature
public class HourRangeTriggerCondition extends ObjectiveLocationTriggerCondition {
```

## Architecture & Concepts
The HourRangeTriggerCondition is a specific, data-driven rule evaluator within the Adventure Mode Objective framework. Its sole responsibility is to determine if the current in-game time falls within a predefined hourly window.

This class is not a service or a manager; it is a passive data container with a single evaluation method. It is designed to be defined in external configuration files (e.g., JSON) and deserialized at runtime by the engine's serialization system, represented by its static CODEC field.

Architecturally, it acts as a leaf node in an objective's condition tree. The core game loop, via an objective management system, polls this condition's `isConditionMet` method to decide if a piece of content should be activated or an objective should progress. It directly queries the server's central timekeeping resource, `WorldTimeResource`, to perform its check, decoupling it from any specific time-management implementation.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's `Codec` system during the deserialization of adventure objective data. The static `CODEC` field on the class defines the mapping between the configuration file keys (MinHour, MaxHour) and the class fields. Manual instantiation is a design violation.
-   **Scope:** The lifetime of an HourRangeTriggerCondition instance is strictly bound to its parent objective configuration. It is loaded into memory when the objective becomes active and persists as long as the objective is available.
-   **Destruction:** The object is marked for garbage collection when its containing objective is unloaded or completed. There are no explicit cleanup or `dispose` methods.

## Internal State & Concurrency
-   **State:** The internal state, consisting of `minHour` and `maxHour`, is effectively **immutable** post-deserialization. While the fields are not declared as final, they are set only once during the codec-driven construction process. The object does not cache any data and holds no references to other non-static game state.
-   **Thread Safety:** The class is **conditionally thread-safe**. The `isConditionMet` method is stateless and performs read-only operations. However, it depends on a `ComponentAccessor` argument, which is a handle to the world state. This method **must** be called from the main server thread or a context where the `ComponentAccessor` is guaranteed to be valid. Invoking it from an asynchronous task with a stale accessor will lead to undefined behavior or crashes.

## API Surface
The public contract is minimal, focused entirely on condition evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConditionMet(accessor, ref, marker) | boolean | O(1) | Evaluates the condition. Returns true if the world's current hour is within the configured range. Correctly handles overnight ranges (e.g., 22:00 to 06:00). |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural Java code. Instead, it is defined declaratively within an objective's configuration file. The engine's objective system will then automatically instantiate and evaluate it.

A conceptual configuration might look like this:

```json
// Example objective_definitions.json
{
  "id": "my_night_quest",
  "triggers": [
    {
      "type": "HourRangeTriggerCondition",
      "MinHour": 20, // 8 PM
      "MaxHour": 5   // 5 AM
    }
  ]
}
```

The engine would then use the `CODEC` to create an `HourRangeTriggerCondition` instance with `minHour = 20` and `maxHour = 5`.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new HourRangeTriggerCondition()`. The object will be uninitialized and useless. All instances must be created via the `Codec` system from a data source.
-   **State Mutation:** Do not attempt to get an instance and modify its `minHour` or `maxHour` fields at runtime. This violates the declarative nature of the system and can lead to unpredictable objective behavior.
-   **External Evaluation:** Do not call `isConditionMet` from systems unrelated to the objective manager. The method requires a valid `ComponentAccessor` from the server's entity-component system, which is not guaranteed to be available in other contexts.

## Data Pipeline
The primary flow for this class is from a static configuration file into a live, evaluatable object in the game engine.

> Flow:
> Adventure Objective JSON File -> Engine Asset Loader -> **Hytale Codec System** -> **HourRangeTriggerCondition Instance** -> ObjectiveManager Evaluation Tick -> Boolean Result

