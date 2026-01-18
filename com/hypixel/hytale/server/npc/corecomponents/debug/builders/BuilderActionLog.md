---
description: Architectural reference for BuilderActionLog
---

# BuilderActionLog

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionLog extends BuilderActionLog {
```

## Architecture & Concepts
The BuilderActionLog class is a component within the server-side NPC asset pipeline. It functions as a deserializer and factory, translating a static data definition from a JSON file into a live, executable game object.

Its primary responsibility is to parse a specific JSON configuration that defines a "log to console" action and use that data to construct an instance of ActionLog. This class is a concrete implementation of the Builder pattern, specialized for a single type of NPC action.

This builder is not intended for direct use by game logic developers. Instead, it is discovered and invoked by a higher-level asset management system during server startup or asset hot-reloading. The use of BuilderSupport and StringHolder indicates its integration into a sophisticated system where action parameters, such as the log message, can be resolved dynamically at runtime using the current ExecutionContext.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading system when it encounters a JSON component of type ActionLog. The `readConfig` method is invoked immediately following instantiation to populate the builder's internal state from the JSON data.

- **Scope:** The BuilderActionLog object is short-lived and transient. Its existence is confined to the asset parsing and compilation phase. It serves as a temporary container for configuration data before the final runtime object is built.

- **Destruction:** The object is eligible for garbage collection as soon as the `build` method has been called and the resulting ActionLog instance has been integrated into the parent NPC behavior asset. It holds no persistent state and is not retained by the runtime system.

## Internal State & Concurrency
- **State:** The class maintains a mutable internal state via the `text` field, which is a StringHolder. This state is populated once during the `readConfig` call. After this configuration step, the state should be treated as effectively immutable.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated, configured, and used within a single-threaded asset loading context. The sequence of instantiation, `readConfig`, and `build` is not atomic and must not be accessed concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionLog | O(1) | Constructs and returns a new ActionLog instance using the configured state. |
| readConfig(JsonElement) | BuilderActionLog | O(k) | Populates the builder's internal state from the provided JSON data. Throws if required fields are missing. |
| getText(BuilderSupport) | String | O(1) | Resolves and returns the configured log message text using the provided runtime context. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component, used by asset tooling. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is handled exclusively by the NPC asset system. A game designer or developer defines the action declaratively in a JSON asset file.

**Example NPC Asset Snippet (JSON):**
```json
{
  "type": "Hytale.NPC.Action.Log",
  "Message": "Player {player.name} has entered the trigger volume."
}
```

The asset system processes this JSON, instantiates a BuilderActionLog, and uses it to create the runtime ActionLog object.

**System-Level Invocation (Pseudo-code):**
```java
// This logic resides deep within the asset loading system.
// Do not replicate this pattern in game code.

JsonElement actionJson = parseActionFromAssetFile();
BuilderActionLog builder = new BuilderActionLog();

// Configure the builder from the asset data
builder.readConfig(actionJson);

// Construct the final runtime object
ActionLog runtimeAction = builder.build(assetBuilderSupport);

// Store the runtimeAction in a behavior tree or state machine
npcBehavior.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of BuilderActionLog using `new`. NPC actions must be defined in JSON assets to ensure they are correctly integrated into the game's content pipeline.

- **State Reconfiguration:** Do not call `readConfig` more than once on a single builder instance. The builder is designed for a single, linear configuration path.

- **Runtime Invocation:** This builder is a loading-time utility. It has no role in the main game loop and should never be referenced or used after the server's asset compilation phase is complete.

## Data Pipeline
The flow of data from configuration to execution is linear and unidirectional, managed by the server's asset systems.

> Flow:
> JSON Asset File -> Asset Manager Parser -> **BuilderActionLog** -> ActionLog Instance -> NPC Behavior Tree -> Game Loop Execution

