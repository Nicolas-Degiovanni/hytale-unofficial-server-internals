---
description: Architectural reference for BuilderActionStorePosition
---

# BuilderActionStorePosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionStorePosition extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionStorePosition class is a factory component within the server-side NPC AI framework. Its sole responsibility is to parse a specific JSON configuration snippet and construct an executable ActionStorePosition object.

This class embodies a key design principle of the AI system: behavior definition through data. Instead of hard-coding AI logic, behaviors are defined declaratively in JSON assets. This builder acts as the bridge between the static JSON data and the live, executable Action objects that form the nodes of an NPC's behavior tree. It is not an action itself, but rather the recipe for creating one.

The core concept it manages is the "slot", a named reference to a memory location within an NPC's execution context where a world position can be stored. This allows AI behaviors to dynamically save and recall locations, such as a patrol point, a player's last known location, or a point of interest.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset parser or factory when it encounters the corresponding action type (e.g., "storePosition") within an NPC behavior asset file. It is never managed by a dependency injection container or service registry.
- **Scope:** Extremely short-lived and transient. An instance exists only for the duration of parsing a single JSON object and building one corresponding Action.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the `build` method returns its `ActionStorePosition` product. It holds no persistent references and is not retained.

## Internal State & Concurrency
- **State:** This object is stateful. It contains a mutable `slot` field of type StringHolder, which is populated by the `readConfig` method. This state is essential for the subsequent `build` call but makes the object non-reentrant.

- **Thread Safety:** **This class is not thread-safe.** Its design assumes a single-threaded asset loading pipeline. The `readConfig` method mutates internal state. Concurrent calls to `readConfig` or `build` on the same instance will result in unpredictable behavior and race conditions. Instances must never be shared across threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionStorePosition instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionStorePosition | O(N) | Parses the input JSON, validating and storing the "Slot" configuration. N is the number of keys in the JSON object. |
| getSlot(BuilderSupport) | int | O(1) | Resolves the configured slot name into its runtime integer index. This is called by the created Action, not externally. |

## Integration Patterns

### Standard Usage
The builder is used in a strict, sequential process during asset deserialization. The caller is responsible for instantiating the builder, configuring it, and then building the final product.

```java
// 1. A parser instantiates the builder
BuilderActionStorePosition builder = new BuilderActionStorePosition();

// 2. The relevant JSON snippet is passed for configuration
// json represents: { "Slot": "lastKnownPlayerPosition" }
builder.readConfig(json);

// 3. The final Action object is built and added to the behavior tree
// builderSupport is a contextual object provided by the asset loader
Action action = builder.build(builderSupport);
behaviorTree.addAction(action);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a builder instance to create multiple actions. Its internal state is mutated by `readConfig`, making it suitable for one-time use only.

- **Incorrect Call Order:** Calling `build` before `readConfig` will result in an unconfigured Action, likely leading to runtime exceptions or undefined behavior when the Action is executed.

- **Direct State Manipulation:** Do not attempt to access or modify the internal `slot` field directly. Rely exclusively on the `readConfig` public API for configuration.

## Data Pipeline
This builder is a critical transformation step in the NPC asset loading pipeline. It converts declarative data into an executable object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderActionStorePosition.readConfig()** -> **BuilderActionStorePosition.build()** -> ActionStorePosition Instance -> NPC Behavior Tree

