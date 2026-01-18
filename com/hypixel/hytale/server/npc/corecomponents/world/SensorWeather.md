---
description: Architectural reference for SensorWeather
---

# SensorWeather

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Component

## Definition
```java
// Signature
public class SensorWeather extends SensorBase {
```

## Architecture & Concepts
The SensorWeather class is a specialized component within the server-side NPC Artificial Intelligence framework. It functions as a conditional gate, or a "Sensor", that allows an NPC's behavior to be dependent on the current weather conditions in the game world.

In the Hytale AI system, Sensors are fundamental building blocks used to construct complex behaviors. They evaluate a specific aspect of the game state and return a boolean result. The SensorWeather component's sole responsibility is to determine if the active weather (e.g., "rain", "snow", "hytale:thunderstorm") matches a predefined list of weather patterns.

This component acts as a bridge between the global World state and an individual NPC's decision-making logic. It is configured within an NPC's asset definition and is invoked by the NPC's active Role or behavior tree during the AI update tick. For example, it can be used to trigger behaviors like "seek shelter during a rainstorm" or "become more aggressive during a magical storm".

## Lifecycle & Ownership
- **Creation:** SensorWeather is not instantiated directly. It is constructed by the NPC asset loading system via its corresponding builder, BuilderSensorWeather. This process occurs when the server loads and parses NPC definition files from disk at startup or during a hot-reload.
- **Scope:** An instance of SensorWeather is scoped to a specific behavior within an NPC's configuration. It is not a global or shared service. Its lifetime is tied to the lifecycle of the NPC Role that owns it.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once the parent NPC or its associated Role is unloaded or destroyed, and no other references to it exist.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains an immutable array of weather strings to match against. Critically, it also contains two mutable fields for performance optimization: prevWeatherIndex and cachedResult. These fields cache the result of the last weather check, preventing redundant string comparisons on every AI tick if the world's weather has not changed.
- **Thread Safety:** **Not Thread-Safe.** The component is designed to be owned and operated by a single NPC's AI context, which is typically updated within a single-threaded game loop. The internal caching mechanism (modifying prevWeatherIndex and cachedResult) is not protected by locks.

**WARNING:** Sharing a single SensorWeather instance across multiple NPCs or accessing it from concurrent threads will lead to race conditions and unpredictable AI behavior. Each NPC behavior must have its own unique instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) / O(1) | Evaluates if the current world weather matches the configured patterns. Complexity is O(1) on cache hit (weather unchanged) and O(N) on cache miss (weather changed), where N is the number of configured weather patterns. |
| getSensorInfo() | InfoProvider | O(1) | Intended for debugging and tooling. Returns null in the current implementation. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by gameplay programmers. It is configured declaratively in NPC asset files and executed automatically by the underlying AI engine. The engine invokes the matches method as part of a behavior tree or state machine evaluation.

A conceptual representation of its use by the engine:

```java
// Executed by an NPC's Role or Behavior Tree processor
boolean canExecuteBehavior = myWeatherSensor.matches(entityRef, currentRole, deltaTime, worldStore);

if (canExecuteBehavior) {
    // Execute weather-specific logic, e.g., run for cover.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SensorWeather()`. The component relies on its builder for proper initialization from asset data. Manual creation will result in a non-functional sensor.
- **State Tampering:** Do not externally read from or write to the internal fields prevWeatherIndex or cachedResult. Modifying this state will corrupt the caching logic and cause incorrect AI decisions.
- **Instance Sharing:** Do not reuse a single SensorWeather instance for multiple, distinct NPC behaviors. The internal cache is specific to one context and is not designed for shared use.

## Data Pipeline
The component consumes data from the world state and transforms it into a boolean decision for the AI system.

> Flow:
> Server World Tick -> World Weather State Update -> `Role.getWorldSupport().getCurrentWeatherIndex()` -> **SensorWeather.matches()** -> `StringUtil.isGlobMatching()` -> Boolean Result -> NPC Behavior Tree Execution

