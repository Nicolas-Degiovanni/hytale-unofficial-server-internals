---
description: Architectural reference for BuilderActionSetStat
---

# BuilderActionSetStat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionSetStat extends BuilderActionBase {
```

## Architecture & Concepts
The **BuilderActionSetStat** class is a foundational component within the server's data-driven NPC behavior system. It functions as a deserializer and factory, responsible for translating a declarative JSON configuration into a concrete, executable game logic object: the **ActionSetStat**.

Its primary architectural role is to act as an intermediary during the asset loading pipeline. It consumes a fragment of a JSON definition file that specifies an NPC action—in this case, modifying an entity's statistic—and prepares an **ActionSetStat** instance. This instance is later executed by the NPC's behavior controller during gameplay.

A key concept in its design is the use of deferred-resolution holders (**AssetHolder**, **FloatHolder**, **BooleanHolder**). The builder does not store the final, resolved values (e.g., the float `100.0`). Instead, it stores a recipe for how to retrieve that value at runtime via the **BuilderSupport** context. This powerful pattern allows NPC designers to define stat modifications using dynamic variables or context-dependent values that are only resolved when the action is actually executed.

## Lifecycle & Ownership
- **Creation:** An instance of **BuilderActionSetStat** is created ephemerally by the server's NPC asset loading system when it encounters an action of this type within an NPC's JSON definition. It is not intended for manual instantiation by game logic developers.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of the parsing and building process for a single NPC action.
- **Destruction:** Once the `build` method is called and the resulting **ActionSetStat** object is returned, the builder has fulfilled its purpose. It holds no persistent state and is immediately eligible for garbage collection. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state is mutable. The `readConfig` method directly modifies the internal **AssetHolder**, **FloatHolder**, and **BooleanHolder** fields. This state represents the uncooked configuration data read from the JSON source.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the single-threaded context of the asset loading pipeline.

    > **Warning:** Concurrent calls to `readConfig` or `build` on the same instance will lead to corrupted state and unpredictable behavior. Treat each instance as a single-use object.

## API Surface
The public API is minimal, focusing exclusively on configuration and object creation. The `get...` methods are not intended for external callers but are used by the **ActionSetStat** object that this builder creates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSetStat | O(1) | Constructs and returns the final **ActionSetStat** runtime object. |
| readConfig(JsonElement) | BuilderActionSetStat | O(k) | Populates the builder's internal state from a JSON source. Throws exceptions on malformed or missing data. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly in gameplay code. It is invoked by higher-level asset management systems. The conceptual flow within such a system is as follows.

```java
// Hypothetical usage within an NPC asset loader
JsonElement actionJson = parseNpcDefinitionFile(".../mob.json");
BuilderActionSetStat builder = new BuilderActionSetStat();

// Configure the builder from the data source
builder.readConfig(actionJson);

// The BuilderSupport provides runtime context (e.g., asset registries)
BuilderSupport support = getBuilderSupportForCurrentContext();

// Create the final, executable action
ActionSetStat executableAction = builder.build(support);

// Add the action to the NPC's behavior tree
npc.getBehaviorController().addAction(executableAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Manual Configuration:** Do not use `new BuilderActionSetStat()` and then attempt to set its internal fields manually. The only supported configuration path is via the `readConfig` method, which ensures proper validation and data handling.
- **Reusing Builder Instances:** Do not reuse a builder instance to create multiple, different actions. While technically possible, it is not the intended design and can lead to subtle bugs if state is not perfectly reset. Always create a new builder for each distinct JSON action block.
- **Storing Builder Instances:** There is no reason to hold a reference to a **BuilderActionSetStat** instance after its `build` method has been called. The resulting **ActionSetStat** is the canonical object to be stored and used.

## Data Pipeline
The **BuilderActionSetStat** is a critical step in the data transformation pipeline that converts static asset definitions into live server objects.

> Flow:
> NPC Definition File (JSON) -> Server Asset Loader -> **BuilderActionSetStat**.readConfig() -> **BuilderActionSetStat**.build() -> **ActionSetStat** Instance -> NPC Behavior Controller -> Game State Mutation

