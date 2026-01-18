---
description: Architectural reference for WeatherTriggerCondition
---

# WeatherTriggerCondition

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.triggercondition
**Type:** Configuration Object

## Definition
```java
// Signature
public class WeatherTriggerCondition extends ObjectiveLocationTriggerCondition {
```

## Architecture & Concepts
The **WeatherTriggerCondition** is a declarative component within the Adventure Mode Objective framework. Its primary function is to determine if the current in-game weather at a specific location satisfies a predefined set of conditions. It acts as a bridge between high-level game design configuration (e.g., an objective that requires "rain" or "snow") and the low-level, real-time state of the game world's weather simulation.

This class is not intended for direct instantiation by developers. Instead, it is deserialized from asset files using its static **CODEC** field. During this deserialization process, it performs a critical optimization: human-readable weather ID strings are converted into a sorted integer array of weather indices. This allows the core evaluation logic in the **isConditionMet** method to use a highly efficient binary search, avoiding costly string comparisons during every game tick evaluation.

It is a specialized implementation of the more generic **ObjectiveLocationTriggerCondition**, inheriting the context of being evaluated at a specific point in the world, typically associated with an **ObjectiveLocationMarker**.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the engine's **Codec** system when loading adventure objective definitions from game asset files. The static **CODEC** field defines the entire deserialization and initialization process.
- **Scope:** The object's lifetime is tightly coupled to its parent objective configuration. It persists in memory as long as the objective is loaded and active.
- **Destruction:** The object is eligible for garbage collection once the adventure objective it belongs to is unloaded or completed, and no further references to it exist. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state, specifically the **weatherIds** and **weatherIndexes** arrays, is populated once during the `afterDecode` phase of its creation lifecycle. After this point, the object's state should be considered immutable. The arrays themselves are not final, but modifying them post-creation is a severe anti-pattern that will lead to undefined behavior.
- **Thread Safety:** This class is inherently thread-safe for its intended use case. The **isConditionMet** method is a pure, read-only operation that does not modify any internal or external state. It is designed to be called from the main server game loop thread, which guarantees safe access to the underlying ECS components like **TransformComponent** and **WeatherResource**.

## API Surface
The public contract is minimal, exposing only the core evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConditionMet(accessor, ref, marker) | boolean | O(log N) | Evaluates if the current weather at the entity's location matches any of the configured weather types. Complexity is dominated by a binary search, where N is the number of valid weather types for this condition. |

## Integration Patterns

### Standard Usage
This class is not used procedurally. It is defined declaratively within a game asset file (e.g., JSON) and consumed by the objective system. The system is responsible for invoking **isConditionMet** at the appropriate time.

A developer would configure this in an asset file, not in Java code. The following is a conceptual example of how the *engine* interacts with a deserialized instance.

```java
// Conceptual Engine Code
// An objective's conditions are evaluated each tick.
Objective objective = loadObjectiveFromAsset("my_rain_quest.json");
WeatherTriggerCondition condition = objective.getTriggerCondition();

// In the game loop, for a specific player (playerRef)
boolean met = condition.isConditionMet(world.getComponentAccessor(), playerRef, locationMarker);

if (met) {
    // Advance the objective
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WeatherTriggerCondition()`. Doing so bypasses the critical **CODEC** logic that populates the **weatherIndexes** array from **weatherIds**. An object created this way will fail with a NullPointerException or produce incorrect results.
- **State Mutation:** Do not attempt to modify the **weatherIds** or **weatherIndexes** arrays after the object has been created. The internal logic relies on the **weatherIndexes** array being sorted for its binary search to function correctly.

## Data Pipeline
The flow of data for this component begins with game assets and ends with a boolean evaluation result that influences the game state.

> Flow:
> Asset File (JSON) -> Engine Codec Deserializer -> **WeatherTriggerCondition** (In-Memory Object) -> Objective System Evaluation -> World State Query (Chunk, Environment, Weather) -> Boolean Result

