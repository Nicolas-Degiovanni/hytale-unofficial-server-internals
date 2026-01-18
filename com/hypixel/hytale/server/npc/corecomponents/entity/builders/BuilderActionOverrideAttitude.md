---
description: Architectural reference for BuilderActionOverrideAttitude
---

# BuilderActionOverrideAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionOverrideAttitude extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionOverrideAttitude class is a key component of the server-side NPC behavior system. It functions as a configuration-driven factory, responsible for translating a specific JSON definition into an executable game logic object.

Its primary role is to act as a deserialization blueprint for the "Override Attitude" action. During the server's asset loading phase, a central configuration manager parses NPC behavior files. When it encounters a definition for this action, it instantiates a BuilderActionOverrideAttitude and uses its `readConfig` method to populate it from the provided JSON data.

This class encapsulates the *definition* of an action, not the action itself. The final, executable logic is created when the `build` method is called, which returns an instance of ActionOverrideAttitude. This separation of configuration from execution is a core pattern in the NPC system, allowing for flexible and data-driven AI behaviors.

The use of `EnumHolder` and `DoubleHolder` for internal state is significant. This pattern abstracts the storage of configuration values, allowing them to be resolved later within a specific `ExecutionContext`. This enables dynamic behaviors where values might depend on game state, NPC properties, or other runtime variables.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level NPC behavior asset parser when an "OverrideAttitude" action is defined in a JSON configuration file. The `readConfig` method is called immediately following instantiation to populate the builder's state.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single action definition from a configuration file.
- **Destruction:** The object is eligible for garbage collection as soon as the `build` method has been called and the resulting `Action` object has been passed to the consuming system (e.g., an NPC's behavior tree). There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** The internal state is mutable during the configuration phase via the `readConfig` method. It stores the target `Attitude` and `duration` within specialized `Holder` objects. Once configured, the state is effectively immutable for the remainder of its brief lifecycle.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, configured, and used within the single-threaded context of the server's asset loading pipeline. Concurrent calls to `readConfig` would lead to a corrupted state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Factory method. Constructs and returns a new `ActionOverrideAttitude` instance. |
| readConfig(JsonElement) | BuilderActionOverrideAttitude | O(N) | Deserializes JSON data into the builder's internal state. N is the number of keys. |
| getAttitude(BuilderSupport) | Attitude | O(1) | Resolves and returns the configured attitude from its internal holder. |
| getDuration(BuilderSupport) | double | O(1) | Resolves and returns the configured duration from its internal holder. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component, indicating if it is ready for production use. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is invoked by the server's asset loading and NPC configuration systems. The typical flow is managed entirely by the engine.

```java
// Conceptual example of engine-level usage
JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");
BuilderActionOverrideAttitude builder = new BuilderActionOverrideAttitude();

// 1. Configure the builder from the data source
builder.readConfig(actionJson);

// 2. Build the final, executable action object
BuilderSupport support = createSupportForNpc(npc);
Action overrideAction = builder.build(support);

// 3. Add the action to the NPC's behavior queue
npc.getBehaviorTree().addAction(overrideAction);
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not use `new BuilderActionOverrideAttitude()` in gameplay code. NPC behaviors should be defined entirely within JSON asset files.
- **State Re-use:** Do not attempt to re-use a single builder instance by calling `readConfig` multiple times. A new builder must be instantiated for each unique action definition being parsed.
- **Premature Access:** Calling `build`, `getAttitude`, or `getDuration` before `readConfig` has been successfully invoked will result in uninitialized state and likely cause a runtime exception.

## Data Pipeline
The class serves as a critical translation step between static configuration data and live, in-game objects.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionOverrideAttitude.readConfig()** -> **BuilderActionOverrideAttitude.build()** -> ActionOverrideAttitude Instance -> NPC Behavior Tree Execution

