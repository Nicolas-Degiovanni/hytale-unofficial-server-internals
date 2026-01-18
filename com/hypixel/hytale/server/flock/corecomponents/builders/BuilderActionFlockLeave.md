---
description: Architectural reference for BuilderActionFlockLeave
---

# BuilderActionFlockLeave

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionFlockLeave extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionFlockLeave class is a server-side component within the NPC AI Asset Pipeline. It functions as a stateless factory, responsible for translating a configuration definition into a concrete, executable game object.

Its specific role is to construct an ActionFlockLeave instance. This class represents the *blueprint* for an action, not the action's execution. During server startup or asset loading, a higher-level parser processes NPC behavior definitions (typically from JSON files). When it encounters an action of this type, it uses this builder to instantiate the corresponding runtime action object. This object is then integrated into the NPC's behavior tree or state machine.

The "Flock" system itself is a high-level AI construct for managing emergent group behaviors. This builder provides the declarative mechanism for an NPC's configured logic to explicitly exit its current flock.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's NPC asset loading system. A central parser or factory identifies the action type from a data file and creates a corresponding builder instance to process it. It is not intended for manual creation.
- **Scope:** Ephemeral. The builder's lifetime is strictly limited to the asset parsing and object construction phase. It exists only to produce a single ActionFlockLeave instance.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method is called and its result is stored by the asset system. It holds no persistent references and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable and Stateless.** This class contains no member fields and its behavior is constant. The `readConfig` method is a no-op, indicating that the "Leave Flock" action is parameter-less. All instances of BuilderActionFlockLeave are functionally identical.
- **Thread Safety:** **Fully Thread-Safe.** Due to its immutable and stateless nature, an instance of this class can be safely accessed by multiple threads without synchronization. However, the asset loading pipeline it operates within is typically a single-threaded process.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionFlockLeave | O(1) | Constructs and returns a new ActionFlockLeave instance. This is the primary factory method. |
| readConfig(JsonElement) | BuilderActionFlockLeave | O(1) | Consumes configuration data. For this action, the method is a no-op as there are no parameters. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable summary for tooling and logs. |
| getLongDescription() | String | O(1) | Returns a detailed description of the action's runtime behavior. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Provides metadata about the feature's stability. Returns Stable. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the asset pipeline. The conceptual flow within the engine is as follows.

```java
// Hypothetical usage by an NPC Asset Parser
// NOTE: This is a conceptual example. Do not replicate.

// 1. Parser identifies an action type, e.g., "hytale:action_flock_leave"
Class<BuilderActionBase> builderClass = findBuilderClassFor("hytale:action_flock_leave");
BuilderActionBase builder = builderClass.newInstance();

// 2. Configure the builder from the JSON data
builder.readConfig(actionJsonData);

// 3. Build the final, executable action object
ActionFlockLeave runtimeAction = ((BuilderActionFlockLeave) builder).build(builderSupport);

// 4. Add the action to the NPC's behavior tree
npcBehavior.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionFlockLeave()` in game logic. The NPC asset system is solely responsible for the lifecycle of builder objects.
- **State Assumption:** Do not attempt to modify the builder or expect it to hold state. It is a stateless factory.
- **Reference Caching:** Do not retain a reference to a builder instance after the `build` method has been called. These objects are transient and should be considered expired post-build.

## Data Pipeline
This builder acts as a transformation step in the data pipeline that converts declarative NPC configuration files into executable server-side AI logic.

> Flow:
> NPC Definition (JSON) -> Asset Parsing Service -> **BuilderActionFlockLeave** -> ActionFlockLeave (Instance) -> NPC Behavior Tree -> AI State Machine (Execution)

