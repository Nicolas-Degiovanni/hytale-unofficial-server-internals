---
description: Architectural reference for BuilderActionCompleteTask
---

# BuilderActionCompleteTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionCompleteTask extends BuilderActionPlayAnimation {
```

## Architecture & Concepts

The BuilderActionCompleteTask is a configuration-driven factory class within the server-side NPC Behavior system. It serves as the bridge between a declarative JSON asset definition and a concrete, executable game logic object. Its primary responsibility is to deserialize a JSON configuration block and use it to construct an instance of ActionCompleteTask.

Architecturally, this class is a specialization of BuilderActionPlayAnimation. This inheritance signifies that "completing a task" is treated as a specific type of animation-playing action. This allows it to reuse all the underlying animation targeting and playback logic from its parent, while layering on the specific semantics of task completion.

A key design pattern employed here is the use of Holder objects, such as BooleanHolder for the *playAnimation* field. This defers the final evaluation of a configuration value until runtime, allowing the behavior to be dynamic based on the game state provided by the BuilderSupport context.

Furthermore, the class enforces design-time contracts. By calling `requireInstructionType`, it constrains its usage to specific parts of an NPC behavior tree, namely within an *Interaction* block. This prevents invalid or nonsensical behavior configurations.

## Lifecycle & Ownership

- **Creation:** An instance of BuilderActionCompleteTask is created by the server's asset loading pipeline when it parses an NPC behavior JSON file and encounters an action of this type. The `readConfig` method is invoked immediately following instantiation to populate the object from the JSON data.

- **Scope:** The object's lifetime is bound to the loaded NPC behavior asset. It is **not** a per-NPC instance object; rather, it is a shared, immutable template for a specific NPC type. It persists in memory as long as the asset is loaded.

- **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding NPC asset or the asset is hot-reloaded.

## Internal State & Concurrency

- **State:** The internal state of this class represents deserialized configuration data. It is populated once during the `readConfig` call and is considered **effectively immutable** thereafter. It does not hold or manage any runtime game state.

- **Thread Safety:** This class is **thread-safe for read operations** after the initial asset loading phase. The `build` method can be safely called by multiple threads (e.g., for multiple NPC instances of the same type) as it does not mutate the builder's internal state.

    **WARNING:** The `readConfig` method is **not thread-safe** and must only be called from the single-threaded asset loading context. Calling it at any other time will lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionCompleteTask | O(1) | Constructs a new runtime ActionCompleteTask instance using the builder's configuration. |
| readConfig(JsonElement) | BuilderActionCompleteTask | O(N) | **Internal Use Only.** Populates the builder from a JSON element. N is the number of properties. |
| isPlayAnimation(BuilderSupport) | boolean | O(1) | Resolves the configured *PlayAnimation* flag against the provided runtime context. |

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this class directly in Java. Instead, they define the action declaratively within an NPC's behavior asset file. The engine handles the instantiation and configuration.

**Example NPC Behavior JSON:**
```json
{
  "type": "Hytale.Interaction",
  "actions": [
    {
      "type": "Hytale.CompleteTask",
      "PlayAnimation": true,
      "Animation": "interaction_success"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Do not use `new BuilderActionCompleteTask()`. The asset pipeline is the sole owner of this object's creation and configuration. Manual instantiation results in an unconfigured and useless object.

- **Manual Configuration:** Do not call `readConfig` from game logic. This method is strictly part of the internal asset deserialization process.

- **State Mutation:** Do not attempt to modify the builder's state after it has been loaded. These objects are designed as immutable templates.

## Data Pipeline

The class functions as a specific step in the data transformation pipeline from static assets to live game objects.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> **BuilderActionCompleteTask** (Deserialized Template) -> `build()` method -> ActionCompleteTask (Runtime Instance) -> NPC Behavior Tree Execution

