---
description: Architectural reference for BuilderActionState
---

# BuilderActionState

**Package:** com.hypixel.hytale.server.npc.corecomponents.statememachine.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionState extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionState class is a foundational component within the server-side NPC (Non-Player Character) asset processing pipeline. It functions as a specialized deserializer and factory, responsible for translating a declarative JSON configuration into a concrete, runtime-ready ActionState object.

This class embodies the Builder pattern to decouple the human-readable, string-based asset definition from the performance-critical, integer-indexed representation required by the NPC state machine during live gameplay. Its sole purpose is to parse a specific JSON block, resolve state name strings into efficient integer indices, and construct the final ActionState.

This builder operates exclusively during the server's asset loading and initialization phase. It is not intended for use in the main game loop or any other runtime context.

## Lifecycle & Ownership
The lifecycle of a BuilderActionState instance is extremely brief and tightly controlled by the asset loading system.

-   **Creation:** Instantiated reflectively by the NPC asset pipeline when it encounters an action of this type within an NPC behavior definition file (e.g., a JSON file). Developers do not create this object manually.
-   **Scope:** Ephemeral. An instance exists only for the duration required to parse a single JSON object and build one ActionState. It does not persist.
-   **Destruction:** The object is immediately eligible for garbage collection after the `build` method returns the final ActionState. It holds no external references and is not managed by any persistent registry.

## Internal State & Concurrency
-   **State:** Highly mutable. The primary function of this class is to accumulate state from a JSON source via the `readConfig` method. Fields such as `state`, `subState`, and `clearState` are populated during this process. This internal state is transient and discarded after the `build` operation.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to operate within a single-threaded asset loading context. Concurrent access, particularly concurrent calls to `readConfig` on the same instance, will result in a corrupted internal state and produce an invalid ActionState object. The asset loading framework must enforce serialized access.

## API Surface
The public API is designed for a simple, two-step process: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderActionState | O(N) | Parses the provided JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| build(BuilderSupport support) | ActionState | O(1) | Consumes the internal state to construct and return a new, immutable ActionState instance. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable summary of the action's purpose, intended for use in development tools. |
| getLongDescription() | String | O(1) | Returns a detailed, human-readable description of the action's purpose and parameters for development tools. |

## Integration Patterns

### Standard Usage
This class is not used directly by gameplay programmers. It is invoked by the server's core asset systems. The conceptual flow is as follows:

```java
// This is a conceptual example of how the asset system uses the builder.
// This code does not appear in standard game logic.

JsonElement actionJson = parseNpcBehaviorFile(".../my_npc.json");
BuilderActionState builder = new BuilderActionState();

// 1. Configure the builder from the data source
builder.readConfig(actionJson);

// 2. Build the final, immutable runtime object
ActionState runtimeAction = builder.build(builderSupport);

// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderActionState()` in game logic. The asset pipeline is the sole owner and creator of these objects.
-   **Instance Caching:** Do not attempt to cache or reuse a BuilderActionState instance. Each instance is purpose-built for a single, unique JSON configuration block. Reusing it will lead to state contamination.
-   **Out-of-Order Calls:** Calling `build` before `readConfig` is an error. It will produce an ActionState with uninitialized or default values, leading to severe and difficult-to-diagnose bugs in NPC behavior.

## Data Pipeline
BuilderActionState serves as a critical translation step in the data flow from disk to the live game engine. It transforms declarative data into an executable instruction for the NPC state machine.

> Flow:
> NPC Behavior JSON File -> Gson Parser -> `JsonElement` -> Asset Pipeline -> **BuilderActionState.readConfig()** -> **BuilderActionState.build()** -> `ActionState` Object -> NPC State Machine Execution

