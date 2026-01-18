---
description: Architectural reference for BuilderActionTriggerSpawners
---

# BuilderActionTriggerSpawners

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTriggerSpawners extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionTriggerSpawners class is a key component of the server-side NPC asset pipeline. It functions as a **configuration parser and factory** for a specific runtime behavior: triggering manual spawn markers in the world.

Its primary responsibility is to translate a declarative JSON configuration block into a concrete, executable `ActionTriggerSpawners` object. This class acts as the bridge between the static data defined in game assets and the dynamic, in-game action system. It is part of a larger factory pattern where the NPC asset loader instantiates the appropriate builder based on the action type specified in the JSON data.

This builder encapsulates the validation logic for the action's parameters, such as ensuring the trigger range is a positive number. It utilizes specialized wrapper types like AssetHolder, DoubleHolder, and IntHolder to manage configuration values that may be resolved later within a specific game context.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading system during server startup or when live-reloading NPC behavior assets. It is created reflectively or via a factory lookup when the system encounters an action of type "TriggerSpawners" in an NPC's JSON definition.
- **Scope:** This object is extremely short-lived. Its scope is confined to the parsing of a single action within a single NPC asset file.
- **Destruction:** The builder instance is eligible for garbage collection immediately after its `build` method is called and the resulting `ActionTriggerSpawners` object is integrated into the NPC's behavior tree. It does not persist in memory.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The `readConfig` method populates the `spawner`, `range`, and `count` fields from the input JSON. This state is held temporarily until the `build` method consumes it to create the final Action object. The state represents a validated configuration, not live game data.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. All operations, particularly `readConfig`, must be performed in a single-threaded context, which is guaranteed by the design of the NPC asset loading pipeline. Concurrent modification would lead to a corrupted or invalid configuration.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns the runtime `ActionTriggerSpawners` instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionTriggerSpawners | O(N) | Parses the JSON data, validates parameters, and populates the internal state. N is the number of keys in the JSON object. |
| getSpawner(BuilderSupport) | String | O(1) | Retrieves the configured spawn marker asset name. Intended for use by the created Action object. |
| getRange(BuilderSupport) | double | O(1) | Retrieves the configured trigger radius. Intended for use by the created Action object. |
| getCount(BuilderSupport) | int | O(1) | Retrieves the configured maximum number of spawners to trigger. Intended for use by the created Action object. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked automatically by the server's asset loading systems. The conceptual flow is as follows.

```java
// Conceptual example of the asset loader's logic.
// Do not write this code yourself.

JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");

// The system instantiates the correct builder for the action type
BuilderActionTriggerSpawners builder = new BuilderActionTriggerSpawners();

// The system configures the builder from the data
builder.readConfig(actionJson);

// The system builds the final, executable action
BuilderSupport support = ...; // Obtain from asset loading context
Action runtimeAction = builder.build(support);

// The runtimeAction is then added to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually create an instance with `new BuilderActionTriggerSpawners()`. The asset pipeline manages the lifecycle of builders.
- **State Re-use:** Do not call `readConfig` on the same builder instance multiple times. Builders are single-use objects designed to configure one action.
- **Premature Build:** Do not call `build` before `readConfig` has been successfully invoked. Doing so will produce an action with uninitialized or default values, leading to unpredictable runtime errors or crashes.

## Data Pipeline
The data flow for this component is linear, transforming static configuration data into a runtime object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderActionTriggerSpawners.readConfig()** -> Validated Internal State -> **BuilderActionTriggerSpawners.build()** -> `ActionTriggerSpawners` Instance -> NPC Behavior Tree

