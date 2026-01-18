---
description: Architectural reference for BuilderActionBeacon
---

# BuilderActionBeacon

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionBeacon extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionBeacon class is a transient, stateful builder responsible for deserializing and validating an NPC's "beacon" action configuration from a JSON asset file. It embodies the Builder pattern, acting as a factory for the runtime `ActionBeacon` object.

Its primary role is to serve as a bridge between the static data representation of an NPC's behavior (defined in JSON) and the live, executable action object used by the server's AI system. It parses configuration keys like `Message`, `Range`, and `TargetGroups`, applying strict validation rules to ensure data integrity before the action is ever instantiated.

A key architectural feature is its use of `Holder` objects (e.g., `StringHolder`, `AssetArrayHolder`). This pattern defers the final resolution of configuration values until the `build` method is called. This allows for dynamic, context-aware values that can be resolved at runtime using the provided `BuilderSupport` context, rather than being limited to static, hardcoded strings or numbers in the JSON file.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionBeacon is created by the server's asset parsing system when it encounters a beacon action definition within an NPC's JSON configuration. It is not intended for direct instantiation by game logic developers.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading phase. It exists only to parse a single JSON block, validate its contents, and produce one `ActionBeacon` instance.
- **Destruction:** After the `build` method is called and the resulting `ActionBeacon` is returned, the builder instance has fulfilled its purpose. It holds no further references and is eligible for standard garbage collection. There are no manual cleanup or `close` operations required.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The `readConfig` method directly mutates the instance's fields (`message`, `range`, `targetGroups`, etc.) as it parses the input JSON. This state represents a one-to-one mapping of a specific JSON configuration block.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use during the asset loading pipeline. Accessing an instance from multiple threads will result in race conditions and a corrupted final `ActionBeacon` object.

## API Surface
The public API is designed for a sequential, two-step process: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderActionBeacon | O(N) | Parses and validates the JSON data, populating the builder's internal state. N is the number of keys in the JSON object. Throws exceptions on validation failure. |
| build(BuilderSupport) | ActionBeacon | O(1) | Constructs and returns the final, immutable `ActionBeacon` runtime object using the previously configured state. |
| getMessage(BuilderSupport) | String | O(1) | Resolves the beacon message using the runtime execution context. |
| getTargetGroups(BuilderSupport) | int[] | O(M) | Resolves the target group asset references into an array of runtime tag set indices. M is the number of target groups. |
| getTargetToSendSlot(BuilderSupport) | int | O(1) | Resolves the target slot name into its corresponding integer ID via the `BuilderSupport` context. |

## Integration Patterns

### Standard Usage
The builder is used internally by the asset loading framework. The conceptual flow is to instantiate, configure from JSON, and immediately build the final action object.

```java
// Conceptual example of framework usage
BuilderActionBeacon builder = new BuilderActionBeacon();

// The framework provides the relevant JSON section and populates the builder
builder.readConfig(npcActionJson);

// The framework provides a runtime context (BuilderSupport) to create the final object
ActionBeacon runtimeAction = builder.build(supportContext);

// The runtimeAction is then added to the NPC's behavior tree
npc.getBehaviorController().addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Game logic developers should never instantiate this class with `new BuilderActionBeacon()`. NPC actions should be defined entirely within data files and loaded by the server.
- **State Re-use:** Do not call `readConfig` multiple times on the same instance or attempt to re-use a builder after `build` has been called. Each builder instance is meant to process exactly one configuration block.
- **Pre-emptive Getters:** Do not call the various `get...` methods before `readConfig` has been successfully executed. The internal state will be uninitialized, leading to default or null values and unpredictable runtime behavior.

## Data Pipeline
The BuilderActionBeacon is a critical step in the NPC action hydration pipeline, transforming static data into a live game object.

> Flow:
> NPC Asset JSON File -> Server Asset Parser -> **BuilderActionBeacon**.readConfig() -> **BuilderActionBeacon**.build() -> `ActionBeacon` Instance -> NPC Behavior Controller

