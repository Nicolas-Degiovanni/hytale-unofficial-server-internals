---
description: Architectural reference for BuilderActionList
---

# BuilderActionList

**Package:** com.hypixel.hytale.server.npc.instructions.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderActionList extends BuilderBase<ActionList> {
```

## Architecture & Concepts
The BuilderActionList is a specialized component within the server's NPC asset loading framework. Its sole responsibility is to deserialize a JSON array of action definitions into a runtime ActionList object. It functions as a factory and parser, translating declarative configuration data into executable game logic.

This class is a concrete implementation of the "Builder" pattern, which is used extensively throughout the NPC system to construct complex objects from configuration files. It does not contain the logic for parsing individual actions itself; instead, it delegates this task by composition to the **BuilderObjectListHelper**. This design isolates the logic for handling a list of objects from the logic of parsing the objects themselves, promoting code reuse and separation of concerns.

It sits at a specific point in the asset pipeline: it is invoked when the primary NPC asset parser encounters a JSON key that corresponds to a list of actions (e.g., an "onInteract" or "onDeath" event handler).

## Lifecycle & Ownership
- **Creation:** Instances of BuilderActionList are created dynamically by the core asset loading system, likely a BuilderManager, when a JSON configuration requires the construction of an ActionList. It is not intended for manual instantiation.
- **Scope:** The lifecycle of a BuilderActionList instance is extremely short and tied to a single parsing operation. It is created, used to process one specific JSON array, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is abandoned and garbage collected once its `build` method has been called and the resulting ActionList has been returned to the calling parser. It holds no persistent references and is not managed by a long-lived registry.

## Internal State & Concurrency
- **State:** The class is stateful. Its primary internal state is the `actions` field, an instance of BuilderObjectListHelper. This helper accumulates the state of the child Action builders as the `readConfig` method processes the input JSON array. The state is mutable and is built up progressively during the parsing phase.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The entire NPC asset loading pipeline is designed to be a single-threaded process. The sequence of `readConfig`, `validate`, and `build` constitutes a single, non-atomic operation that modifies internal state. Concurrent access would lead to a corrupted or unpredictable ActionList.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionList | O(N) | Finalizes the build process. Consumes the internal state to construct and return the immutable ActionList. Returns an empty list if no actions were parsed. |
| readConfig(JsonElement) | BuilderActionList | O(N) | Parses a JSON array, delegating the creation of individual Action builders to the internal helper. This is the primary entry point for populating the builder's state. |
| validate(...) | boolean | O(N) | Recursively invokes validation on all child Action builders that have been parsed. Aggregates errors into the provided list. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked automatically by the higher-level asset loading framework. A parent builder, such as one for an NPC's personality or behavior tree, would delegate the parsing of an "actions" block to this class.

```java
// Conceptual usage within a parent builder
// The framework handles the instantiation and invocation of BuilderActionList
// when it encounters the "onDeathActions" key in the JSON.

// someNpcConfig.json
{
  "onDeathActions": [
    { "type": "playSound", "sound": "mob.zombie.death" },
    { "type": "dropLoot", "table": "zombie_common" }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionList()`. The asset framework manages the lifecycle of builders. Manual creation bypasses necessary context and management provided by the BuilderManager.
- **Instance Reuse:** A BuilderActionList instance is stateful and designed for a single `readConfig` -> `validate` -> `build` cycle. Attempting to reuse an instance to parse a second JSON array will result in corrupted state, blending actions from both sources.
- **Out-of-Order Calls:** The `build` method must only be called after `readConfig` has successfully populated the builder. Calling it on a fresh instance will produce an empty ActionList without any diagnostic error.

## Data Pipeline
The BuilderActionList is a critical step in transforming static configuration data into live server objects.

> Flow:
> NPC JSON File -> GSON Parser -> `JsonElement` (Array) -> **BuilderActionList.readConfig** -> `BuilderObjectListHelper` -> **BuilderActionList.build** -> `ActionList` Object -> NPC Behavior Component

