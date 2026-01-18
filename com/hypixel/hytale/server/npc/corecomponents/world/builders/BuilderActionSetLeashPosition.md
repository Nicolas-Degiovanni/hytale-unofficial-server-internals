---
description: Architectural reference for BuilderActionSetLeashPosition
---

# BuilderActionSetLeashPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderActionSetLeashPosition extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetLeashPosition class is a factory component within the server's data-driven NPC behavior system. It is not a runtime action itself; rather, it is responsible for parsing a JSON configuration snippet and constructing an executable **ActionSetLeashPosition** instance.

This class embodies the Builder Pattern. It serves as a translation layer between a designer-authored JSON file and a concrete, in-engine game logic object. Its primary role is to validate configuration, declare dependencies via the BuilderSupport context, and ultimately instantiate the final action. Each `BuilderAction...` class in this package corresponds to a specific action type that can be defined in an NPC's behavior asset files.

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the server's NPC asset loading pipeline. When the pipeline encounters an action of this type in a JSON file, it creates a new BuilderActionSetLeashPosition instance and immediately calls the readConfig method.
-   **Scope:** Ephemeral and extremely short-lived. An instance exists only for the brief period required to parse a single JSON object and build the corresponding ActionSetLeashPosition object.
-   **Destruction:** The builder object is intended to be discarded and becomes eligible for garbage collection immediately after the `build` method returns. It holds no persistent state and is not registered in any system-wide context.

## Internal State & Concurrency
-   **State:** The internal state consists of mutable boolean flags, `toTarget` and `toCurrent`. This state is populated exclusively by the readConfig method and is used only during the subsequent call to `build`. The object does not cache any data.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single, synchronous asset loading thread. Do not share instances of this builder across threads or invoke its methods concurrently.

## API Surface
The public API is designed for use by the asset loading system, not for general-purpose game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSetLeashPosition | O(1) | Constructs and returns the final ActionSetLeashPosition runtime object. |
| readConfig(JsonElement) | BuilderActionSetLeashPosition | O(N) | Parses the JSON definition, populating the builder's internal state. N is the number of keys. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable description, likely for use in development tools. |
| getLongDescription() | String | O(1) | Returns a detailed, human-readable description for development tools or documentation. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. It is invoked by the underlying NPC behavior asset system. The conceptual flow within that system is as follows.

```java
// Conceptual example from the asset loading system
JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");
BuilderActionSetLeashPosition builder = new BuilderActionSetLeashPosition();

// 1. Configure the builder from the data file
builder.readConfig(actionJson);

// 2. Build the final runtime action
BuilderSupport supportContext = new BuilderSupport();
ActionSetLeashPosition runtimeAction = builder.build(supportContext);

// 3. Add the action to the NPC's behavior tree
npcBehaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Configuration:** Creating an instance with `new BuilderActionSetLeashPosition()` and calling `build` without first calling `readConfig` will produce a default, likely useless, action object. The builder's state will be uninitialized.
-   **Instance Reuse:** Do not retain and reuse a builder instance to create multiple actions. Each builder is designed for a one-shot translation of a single JSON definition.

## Data Pipeline
This builder acts as a transformation step in the NPC asset loading pipeline. It converts declarative data into an executable object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderActionSetLeashPosition** (`readConfig` & `build`) -> ActionSetLeashPosition Instance -> NPC Behavior Tree

