---
description: Architectural reference for BuilderActionAppearance
---

# BuilderActionAppearance

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionAppearance extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionAppearance class is a transient, configuration-time component within the server's NPC Action System. Its primary architectural role is to serve as a data-to-object translator, specifically converting a declarative JSON configuration into an executable runtime object, the ActionAppearance.

This class is not a long-lived service. Instead, it is a single-use builder that embodies the "fail-fast" design principle. During the server's asset loading phase, it is instantiated to parse a specific JSON block. It immediately validates the provided configuration, such as ensuring the specified model asset actually exists via the ModelExistsValidator. This prevents invalid NPC definitions from ever reaching the runtime environment, thereby increasing server stability.

It acts as a critical link between the static data assets that define NPC behavior and the dynamic, in-game action system that executes that behavior.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level factory or parser, likely when an action of type "appearance" is encountered within an NPC's JSON definition file. It is never managed by a dependency injection container or service locator.
- **Scope:** Extremely short-lived. An instance of BuilderActionAppearance exists only for the duration of parsing one specific action definition. Its lifecycle is confined to the call stack of the asset loading process.
- **Destruction:** The object becomes eligible for garbage collection immediately after its `build` method is invoked and the resulting ActionAppearance object is returned to the caller. It holds no persistent state and is not registered in any system-wide context.

## Internal State & Concurrency
- **State:** The internal state is mutable, consisting of a single `appearance` string. This field is populated during the configuration phase by the `readConfig` method. The object is inherently stateful but only for its brief existence.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use during the server's boot or asset-loading sequence. Concurrent access would lead to race conditions and unpredictable configuration states.

## API Surface
The public API is minimal, reflecting its focused role as a builder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionAppearance | O(1) | Constructs and returns the final ActionAppearance runtime object. This is the terminal operation. |
| readConfig(JsonElement) | BuilderActionAppearance | O(1) | Parses the JSON input, validates asset references, and populates the builder's internal state. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable summary of the action for use in tooling or logs. |
| getLongDescription() | String | O(1) | Returns a detailed, human-readable description of the action for use in tooling. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the server's internal NPC definition parser. The pattern is to instantiate, configure, and build in immediate succession.

```java
// Pseudo-code representing the factory's logic
JsonElement actionConfig = parseNpcDefinitionFile(".../npc.json");

// 1. A new builder is created for the specific action block
BuilderActionAppearance builder = new BuilderActionAppearance();

// 2. The builder is configured from the data source
builder.readConfig(actionConfig);

// 3. The final, immutable action object is created
ActionAppearance runtimeAction = builder.build(builderSupport);

// The builder is now discarded and the runtimeAction is attached to the NPC
npc.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of BuilderActionAppearance for later use. Each action definition parsed from JSON **must** use a new builder instance to ensure state isolation.
- **Premature Build:** Do not call `build` before `readConfig` has been successfully invoked. Doing so will produce an improperly configured ActionAppearance object, leading to a NullPointerException or other undefined behavior at runtime.
- **External Instantiation:** Game logic or plugin code should never instantiate this class directly. It is an internal implementation detail of the asset loading pipeline.

## Data Pipeline
This component operates exclusively within the server's configuration and asset loading pipeline. It does not participate in the main game loop.

> Flow:
> NPC Definition (*.json file*) -> JSON Parser -> NPC Action Factory -> **BuilderActionAppearance** -> ActionAppearance (runtime object) -> Attached to NPC Entity

