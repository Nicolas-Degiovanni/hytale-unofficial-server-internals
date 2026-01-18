---
description: Architectural reference for BuilderActionReleaseTarget
---

# BuilderActionReleaseTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionReleaseTarget extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionReleaseTarget is a factory component within the server's NPC behavior asset pipeline. Its sole responsibility is to translate a declarative JSON configuration into a runtime-executable `ActionReleaseTarget` object.

This class is not part of the core game loop. Instead, it operates exclusively during the server's asset loading and parsing phase. It acts as an intermediary, decoupling the static data representation (JSON) of an NPC action from its live, in-memory object representation. Each "release target" action defined in an NPC's behavior file will cause a corresponding instance of this builder to be created and configured.

The use of a `StringHolder` for `targetSlot` is a key design choice. It defers the resolution of the target slot name (e.g., "LockedTarget") into a numerical ID until the `build` phase, allowing the system to validate and link identifiers within a broader execution context provided by `BuilderSupport`.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset parsing system when it encounters an action of this type within an NPC's JSON definition. It is never created manually during gameplay.
- **Scope:** Extremely short-lived. An instance exists only for the duration required to parse a single JSON object and build one `ActionReleaseTarget` instance.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method returns its product. It holds no persistent references and is not registered in any long-term registry.

## Internal State & Concurrency
- **State:** Mutable. The primary state is the `targetSlot` field, which is populated by the `readConfig` method. This state is transient and serves only as a temporary container for configuration data before the final `ActionReleaseTarget` object is constructed.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the asset loading sequence. Concurrent access would result in a race condition, corrupting the configuration state of the builder.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionReleaseTarget | O(1) | Constructs and returns the final, immutable `ActionReleaseTarget` object. |
| readConfig(JsonElement) | BuilderActionReleaseTarget | O(1) | Configures the builder's internal state from a JSON data source. This is the primary entry point for deserialization. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot string into its runtime integer ID using the provided context. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal NPC asset loading system. The pattern is always to instantiate, configure from JSON, and then build the final action object.

```java
// Pseudocode for engine's asset parser
BuilderActionReleaseTarget builder = new BuilderActionReleaseTarget();

// Configure the builder from the parsed JSON data
builder.readConfig(jsonActionElement);

// Construct the runtime action, passing in the necessary context
ActionReleaseTarget runtimeAction = builder.build(builderSupportContext);

// Add the runtimeAction to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a `BuilderActionReleaseTarget` instance to build multiple actions. Its internal state is specific to a single `readConfig` call. A new builder must be instantiated for each action defined in the configuration.
- **Manual Instantiation:** Never instantiate this class in gameplay logic. It is strictly a config-time component. Interacting with NPC behaviors at runtime should be done through the NPC's AI controller, not by creating new builder instances.

## Data Pipeline
The builder is a critical step in the data transformation pipeline that converts static NPC definitions into live server-side entities.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderActionReleaseTarget**.readConfig() -> **BuilderActionReleaseTarget**.build() -> `ActionReleaseTarget` Runtime Object -> NPC Behavior Tree

