---
description: Architectural reference for BuilderActionIgnoreForAvoidance
---

# BuilderActionIgnoreForAvoidance

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionIgnoreForAvoidance extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionIgnoreForAvoidance class is a factory component within the server-side NPC behavior system. It adheres to the Builder pattern, serving the specific purpose of translating a static data definition (JSON) into a live, executable game object.

Its primary role is to parse a configuration for and subsequently construct an instance of ActionIgnoreForAvoidance. This action instructs an NPC's pathfinding and avoidance systems to disregard a specific entity, which is identified by a "target slot". This class acts as the critical link between the declarative NPC behavior assets authored by designers and the imperative code that executes within the game loop. It is a key component of the asset hydration pipeline, ensuring that abstract definitions are converted into concrete, runtime-ready instructions.

## Lifecycle & Ownership
- **Creation:** Instances are created dynamically by the NPC asset loading framework. When the framework parses an NPC behavior tree from a JSON file and encounters an action of this type, it instantiates a new BuilderActionIgnoreForAvoidance to handle the configuration.
- **Scope:** The lifecycle of a builder instance is extremely short and confined to the asset loading process. It exists only to configure and build a single ActionIgnoreForAvoidance object.
- **Destruction:** Once the build method has been called and the resulting Action is integrated into the NPC's behavior tree, the builder object is no longer referenced and becomes eligible for garbage collection. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class maintains mutable state. The primary state field is targetSlot, a StringHolder that stores the name of the target slot read from the JSON configuration. This state is populated by the readConfig method and later resolved into an integer ID by getTargetSlot.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for use within the single-threaded context of the asset loading pipeline. Its methods, particularly readConfig, mutate internal state, which would lead to race conditions and unpredictable behavior if accessed concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionIgnoreForAvoidance instance using the internal state populated by readConfig. |
| readConfig(JsonElement) | BuilderActionIgnoreForAvoidance | O(1) | Parses the provided JSON data to configure the builder's internal state. Requires a "TargetSlot" string property in the JSON. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its runtime integer identifier using the provided BuilderSupport context. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon for most developers. It is invoked automatically by the server's asset loading systems. The conceptual flow within the engine is as follows.

```java
// Engine-level code (conceptual)
// The engine finds the appropriate builder for a JSON block.
BuilderActionIgnoreForAvoidance builder = new BuilderActionIgnoreForAvoidance();

// The engine passes the JSON data to configure the builder.
builder.readConfig(actionJsonData);

// The engine provides a support context and builds the final action.
Action finalAction = builder.build(engineSupportContext);

// The finalAction is then added to the NPC's behavior tree.
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not attempt to reuse a single builder instance to create multiple actions. Each builder is stateful and intended for a single build operation.
- **Build Before Configure:** Calling build before readConfig will result in an improperly configured Action, likely causing runtime errors or NullPointerExceptions when the action is executed.
- **Manual Instantiation:** Avoid `new BuilderActionIgnoreForAvoidance()` in general game logic. The creation and configuration of these builders is the exclusive responsibility of the NPC asset pipeline.

## Data Pipeline
This builder acts as a transformation step in the data pipeline that converts static assets into executable NPC logic.

> Flow:
> NPC Behavior JSON Asset -> Asset Deserializer -> **BuilderActionIgnoreForAvoidance.readConfig()** -> **BuilderActionIgnoreForAvoidance.build()** -> ActionIgnoreForAvoidance Instance -> NPC Runtime Behavior Tree

