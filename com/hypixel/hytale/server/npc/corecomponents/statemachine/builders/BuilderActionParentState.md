---
description: Architectural reference for BuilderActionParentState
---

# BuilderActionParentState

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionParentState extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionParentState class is a configuration-time component that acts as a factory for the runtime ActionParentState. It is not part of the live game simulation loop. Instead, its sole purpose is to parse a specific block of a JSON asset file and translate that declarative configuration into an executable game object.

This builder is a critical link in the NPC asset pipeline. It allows game designers to define, in a simple JSON format, an action that can force a change in an NPC's top-level state machine from within a nested behavior or component. For example, a specific attack animation component could use this action to transition the entire NPC from a "Combat" state to a "Fleeing" state.

It operates within a larger framework, indicated by its extension of BuilderActionBase and its reliance on the BuilderSupport context object. This framework is responsible for orchestrating the parsing of entire NPC definitions, with this class handling one specific, specialized instruction type.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderActionParentState is created dynamically by a higher-level asset parsing service. This occurs when the service encounters a JSON object whose type identifier maps to this specific builder. It is never instantiated directly by gameplay logic.
- **Scope:** The object's lifetime is extremely short, confined entirely to the asset loading and validation phase. It exists only long enough to process its corresponding JSON element and produce a final ActionParentState object via the build method.
- **Destruction:** The builder instance is immediately eligible for garbage collection after the build method returns. The ownership of the created ActionParentState object is transferred to the parent NPC asset being constructed.

## Internal State & Concurrency
- **State:** This class is stateful but only during its brief lifecycle. Its primary internal state is the `state` string, which holds the name of the target state to transition to. This field is populated and validated by the readConfig method.

- **Thread Safety:** **Not thread-safe.** This class is designed exclusively for single-threaded execution during the server's synchronous asset loading process. Attempting to share an instance across threads or call its methods concurrently will result in race conditions and unpredictable state corruption. The entire asset builder system assumes a single-threaded, deterministic environment.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionParentState | O(1) | Constructs and returns the final, immutable ActionParentState runtime object. Throws if called before readConfig. |
| readConfig(JsonElement) | BuilderActionParentState | O(N) | Configures the builder by parsing a JSON element. Validates the target state name against the NPC's defined states. |
| getStatePair(BuilderSupport) | StatePair | O(log M) | Resolves the configured state string into an internal, optimized StatePair representation using the BuilderSupport context. |

## Integration Patterns

### Standard Usage
This class is intended to be used exclusively by the server's asset loading systems. A new builder is instantiated for each corresponding JSON action, configured once, and then used to build the final runtime object.

```java
// Conceptual usage within a hypothetical asset loader
JsonElement actionJson = ... // A JSON block for this action
BuilderSupport supportContext = ... // The context for the current NPC asset

// 1. Instantiate the builder
BuilderActionParentState builder = new BuilderActionParentState();

// 2. Configure from the data source
builder.readConfig(actionJson);

// 3. Build the final runtime action
ActionParentState runtimeAction = builder.build(supportContext);

// 4. Integrate the action into the NPC's behavior
npcBehavior.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Runtime Instantiation:** Never create an instance of BuilderActionParentState during the game loop. It is a configuration-time tool, not a runtime component.
- **Builder Reuse:** Do not attempt to reuse a single builder instance to parse multiple JSON blocks. This will cause state from the previous configuration to leak into the next, leading to incorrect behavior. Always create a new instance for each action.
- **Premature Build:** Calling the build method before readConfig has been successfully executed will result in an unconfigured or partially configured ActionParentState, likely causing a NullPointerException or other runtime assertion failures.

## Data Pipeline
This builder acts as a transformation step in the data pipeline that converts static asset files into live server objects.

> Flow:
> NPC Definition (*.json file*) -> JSON Parsing Library -> Asset Factory Service -> **BuilderActionParentState.readConfig()** -> **BuilderActionParentState.build()** -> ActionParentState (Runtime Object) -> NPC State Machine Component

