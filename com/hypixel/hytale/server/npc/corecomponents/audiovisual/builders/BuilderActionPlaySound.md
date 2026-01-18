---
description: Architectural reference for BuilderActionPlaySound
---

# BuilderActionPlaySound

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Configuration Factory

## Definition
```java
// Signature
public class BuilderActionPlaySound extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionPlaySound class is a configuration-time factory within the server-side NPC Behavior System. It adheres to the Builder pattern, serving a single, critical purpose: to translate a static data definition from an asset file (typically JSON) into a live, executable game object, ActionPlaySound.

This class is not used during the active game loop. Instead, it operates during the server's asset loading and initialization phase. It is responsible for parsing the configuration for a "play sound" action, validating the specified sound asset reference, and ultimately constructing the runtime ActionPlaySound component. This strict separation between configuration (the Builder) and execution (the Action) is a core design principle of the NPC system, allowing for robust validation and efficient runtime performance.

The builder encapsulates all logic related to deserialization and asset validation. For example, it uses a SoundEventExistsValidator to ensure that the sound event ID specified in the configuration corresponds to a real, loaded asset. This prevents entire classes of runtime errors by catching invalid data at server startup.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset parsing system when an action of this type is encountered within an NPC's behavior definition file. The `readConfig` method is invoked immediately following instantiation to populate the builder from the source JSON data.
-   **Scope:** The object's lifecycle is ephemeral and confined to the asset loading phase. It exists only to parse its configuration and produce an ActionPlaySound instance via the `build` method.
-   **Destruction:** Once the `build` method is called and the resulting ActionPlaySound object is registered with the parent NPC's behavior component, the BuilderActionPlaySound instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** The primary internal state is the `soundEventId`, stored within an AssetHolder. This field is mutable but is designed to be written to exactly once during the `readConfig` call. After initialization, the state is effectively immutable for the remainder of the object's short lifecycle.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single thread during the server's synchronous asset loading process. Concurrent access, especially to the `readConfig` method, would result in a race condition and corrupt the builder's state. All interaction with this class must be externally synchronized if used in a multi-threaded loading environment.

## API Surface
The public API is focused on the object's role as a factory and data resolver.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionPlaySound | O(1) | Constructs and returns the runtime ActionPlaySound object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionPlaySound | O(N) | Deserializes configuration from a JSON source. Populates and validates internal state. Throws if required data is missing or invalid. |
| getSoundEventId(BuilderSupport) | String | O(1) | Resolves and returns the string identifier for the configured sound event (e.g., "hytale:mobs.skeleton.step"). |
| getSoundEventIndex(BuilderSupport) | int | O(log M) | **Performance Critical.** Resolves the string ID to a unique integer index via the global SoundEvent.AssetMap. This is used by the runtime to avoid expensive string operations. Throws IllegalArgumentException if the key is not found post-initialization. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly in code. Instead, they define the action declaratively in an NPC's JSON asset file. The engine's asset loader uses this builder internally.

The conceptual system-level usage is as follows:

```java
// This code is representative of the NPC Asset Loader's internal logic.
// Do not invoke this class directly.

JsonElement actionJson = parseNpcAssetFile(".../skeleton.json");

// 1. The system instantiates the builder
BuilderActionPlaySound builder = new BuilderActionPlaySound();

// 2. The system configures it from the asset data
builder.readConfig(actionJson);

// 3. The system builds the runtime component
BuilderSupport support = createBuilderSupportForNpc();
ActionPlaySound runtimeAction = builder.build(support);

// 4. The runtime action is added to the NPC's behavior tree
npc.getBehaviorTree().addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BuilderActionPlaySound()`. The object is useless without being configured by the asset pipeline via `readConfig`.
-   **State Mutation After Read:** Do not attempt to modify the builder's state after `readConfig` has been called. The object is not designed for reuse or modification.
-   **Premature Index Resolution:** Calling `getSoundEventIndex` before `readConfig` will result in an error, as the internal `soundEventId` will not have been populated.

## Data Pipeline
The BuilderActionPlaySound serves as a key transformation step in the NPC audio data pipeline, converting declarative configuration into an optimized, runtime-ready format.

> Flow:
> NPC JSON Asset -> Server Asset Loader -> **BuilderActionPlaySound.readConfig** -> **BuilderActionPlaySound.build** -> ActionPlaySound (Instance) -> NPC Behavior Tree -> Game Loop Trigger -> Audio Engine Request (using integer index)

