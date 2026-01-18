---
description: Architectural reference for BuilderActionSetAlarm
---

# BuilderActionSetAlarm

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSetAlarm extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetAlarm class is a transient factory object within the server-side NPC asset system. It is not the action itself, but rather the component responsible for deserializing and constructing an immutable ActionSetAlarm instance from a JSON configuration.

Its primary role is to act as a bridge between the declarative data format (JSON) that defines an NPC's behavior and the executable, in-memory representation of that behavior. This class is one of many specialized builders, each corresponding to a specific action type an NPC can perform.

A key architectural pattern employed here is the use of Holder objects, such as StringHolder and TemporalArrayHolder. These wrappers decouple the static configuration from the dynamic runtime context. This allows the JSON to define a placeholder for a value (e.g., an alarm name) which is then resolved against the NPC's current state or execution context when the action is actually built or executed. This provides significant flexibility, enabling behaviors to be data-driven and context-aware without hardcoding values.

This builder is invoked by a higher-level asset parser which identifies the action type from the JSON and delegates the construction process to the appropriate builder.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading pipeline when it encounters a JSON object with a type field corresponding to "SetAlarm". It is never created directly by game logic developers.
- **Scope:** The lifecycle of a BuilderActionSetAlarm instance is extremely short. It exists only for the duration of parsing a single JSON action block.
- **Destruction:** After the `build` method is called and the resulting ActionSetAlarm object is returned to the asset loader, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist in the game state.

## Internal State & Concurrency
- **State:** The object is highly mutable during its lifecycle. The `readConfig` method directly mutates the internal `name` and `durationRange` fields. This state represents the parsed, but not yet fully resolved, configuration from the source JSON.
- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be used in a single-threaded context during asset deserialization. Concurrent access would lead to a corrupted or unpredictable internal state. This design is safe under the assumption that a single NPC behavior asset is processed synchronously.

## API Surface
The public API is designed for a sequential, two-step process: configuration followed by construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSetAlarm | O(1) | Constructs the final, immutable ActionSetAlarm instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionSetAlarm | O(N) | Populates the builder's internal state from a JSON element. N is the number of keys. Returns `this` to support chaining. |
| getAlarm(BuilderSupport) | Alarm | O(1) | Resolves the configured alarm name against the provided context to fetch a runtime Alarm object. |
| getDurationRange(BuilderSupport) | TemporalAmount[] | O(1) | Resolves the configured duration range against the provided context. |

## Integration Patterns

### Standard Usage
The builder is used exclusively by the NPC asset loading system. The pattern is to instantiate, configure, and build immediately.

```java
// Pseudo-code for asset loading
JsonElement actionJson = getActionConfigFromJson();
BuilderActionSetAlarm builder = new BuilderActionSetAlarm();

// 1. Configure the builder from the data source
builder.readConfig(actionJson);

// 2. Build the final, immutable action object using runtime context
ActionSetAlarm action = builder.build(builderSupport);

// The 'action' is now stored in the NPC's behavior tree or action list.
// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Game logic developers should never instantiate this class with `new`. The NPC behavior system manages the creation of actions from data files. Modifying NPC behavior should be done by editing the source JSON assets, not by constructing actions in code.
- **State Re-use:** Do not attempt to reuse a builder instance by calling `readConfig` multiple times. The builder is not designed to be reset and may carry over state from previous configurations, leading to undefined behavior.
- **Post-Build Modification:** Calling any method on the builder after `build` has been called is an error. The builder's lifecycle is complete at that point.

## Data Pipeline
This builder is a critical step in the pipeline that transforms static data into a live NPC behavior component.

> Flow:
> NPC_Behavior.json -> GSON Parser -> **BuilderActionSetAlarm.readConfig()** -> **BuilderActionSetAlarm.build()** -> ActionSetAlarm Instance -> NPC Behavior Tree

