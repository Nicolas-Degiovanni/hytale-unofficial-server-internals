---
description: Architectural reference for BuilderActionPlayAnimation
---

# BuilderActionPlayAnimation

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderActionPlayAnimation extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionPlayAnimation class is a configuration-time component within the server's NPC (Non-Player Character) asset system. It serves as a factory for creating runtime *Action* objects from static data definitions, typically sourced from JSON files.

Its primary responsibility is to deserialize a JSON configuration block that specifies an animation and translate it into a concrete ActionPlayAnimation instance. This class embodies the data-driven design principle for NPC behaviors, allowing designers to define complex actions in data files without modifying engine code.

This builder is part of a larger polymorphic system where different `BuilderAction...` classes are responsible for different types of NPC actions. The parent class, BuilderActionBase, provides the common contract for this system. This builder specifically handles the "PlayAnimation" action type.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionPlayAnimation is created by the server's NPC asset loading pipeline. When the loader parses an NPC behavior graph and encounters an action node of type "PlayAnimation", it instantiates this class to process that node's configuration.

- **Scope:** The object's lifetime is extremely short and confined to the asset loading phase. It exists only to parse its corresponding JSON block, validate the inputs, and produce a final ActionPlayAnimation object via its `build` method.

- **Destruction:** Once the `build` method has been called and the resulting ActionPlayAnimation object is integrated into the NPC's runtime behavior tree, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the main game loop.

## Internal State & Concurrency
- **State:** The internal state is mutable and populated by the `readConfig` method. It holds the target animation slot (e.g., `FullBody`, `UpperBody`) and the animation asset identifier. The animation identifier is stored in a StringHolder, which allows for dynamic expression evaluation at runtime, enabling animations to be selected based on game state.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed exclusively for use within the single-threaded context of the asset loading system. This is an intentional design choice, as the builder's lifecycle is short and does not overlap with the multi-threaded game loop.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionPlayAnimation | O(1) | Constructs and returns the runtime ActionPlayAnimation object. |
| readConfig(JsonElement) | BuilderActionPlayAnimation | O(1) | Deserializes JSON data into the builder's internal state. Throws exceptions on malformed input. |
| runLoadTimeValidationHelper0(...) | void | O(N) | Hooks into the asset validation pipeline to verify that the specified animation ID exists. |
| getSlot() | NPCAnimationSlot | O(1) | Returns the configured animation slot. |
| getAnimationId(BuilderSupport) | String | O(k) | Resolves and returns the animation ID, potentially evaluating an expression via the ExecutionContext. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the NPC asset loading system. The conceptual usage pattern is as follows.

```java
// Conceptual example within the asset loading system
JsonElement configData = parseNpcAction("..."); // Reads a JSON block
BuilderActionPlayAnimation builder = new BuilderActionPlayAnimation();

// 1. Configure the builder from data
builder.readConfig(configData);

// 2. Validate the configuration at load-time
builder.runLoadTimeValidationHelper0(...);

// 3. Build the final runtime object
ActionPlayAnimation runtimeAction = builder.build(builderSupport);

// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Runtime Instantiation:** Never create an instance of this class during the main game loop. It is a load-time-only utility.
- **State Re-use:** Do not attempt to reuse a builder instance to configure multiple actions. Each builder should be instantiated, configured, and used to build exactly one action object before being discarded.
- **Post-Build Modification:** Modifying the state of the builder after calling `build` has no effect on the already-created ActionPlayAnimation object.

## Data Pipeline
The builder acts as a transformation step, converting declarative configuration data into an executable runtime object.

> Flow:
> NPC Definition JSON File -> Server Asset Loader -> **BuilderActionPlayAnimation** (`readConfig`) -> `build()` -> ActionPlayAnimation (Runtime Object) -> NPC Behavior State Machine -> Animation System Execution

