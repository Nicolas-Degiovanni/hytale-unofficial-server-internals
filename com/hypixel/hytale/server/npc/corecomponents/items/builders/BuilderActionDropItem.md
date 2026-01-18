---
description: Architectural reference for BuilderActionDropItem
---

# BuilderActionDropItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items.builders
**Type:** Transient / Configuration Builder

## Definition
```java
// Signature
public class BuilderActionDropItem extends BuilderActionWithDelay {
```

## Architecture & Concepts
The BuilderActionDropItem is a configuration-time component responsible for deserializing and validating the parameters for an NPC "drop item" action. It functions as a key part of the server's NPC Behavior system, which translates declarative JSON configuration files into executable, in-game logic.

This class embodies the Builder pattern. Its primary role is to act as an intermediary between a raw JSON definition and a concrete, runtime Action object. During the server's asset loading phase, a central factory identifies an action of type "DropItem" in an NPC's behavior file and instantiates a BuilderActionDropItem to process it. The builder consumes the JSON, populates its internal fields, performs critical load-time validation, and finally, produces an immutable ActionDropItem instance.

This architecture decouples the complex logic of parsing, validation, and default value assignment from the runtime execution object. The builder is a transient, single-use object that is discarded once the final Action is constructed, ensuring that the game loop only deals with clean, validated, and efficient action objects.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system when it encounters a "DropItem" action definition within an NPC's JSON configuration. It is never created directly during the game loop.
-   **Scope:** Extremely short-lived. Its existence is confined to the duration of the `readConfig` and `build` methods during the server's startup or asset-reload sequence.
-   **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method returns the final ActionDropItem instance. No references to the builder are maintained post-configuration.

## Internal State & Concurrency
-   **State:** Highly mutable. The object is created in a default state, and its fields (item, dropList, throwSpeed, etc.) are populated sequentially by the `readConfig` method. After this method completes, its state is effectively frozen and is used as a data source for the `build` method. It caches the deserialized configuration for a single action.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within the asset loading pipeline. Accessing a single instance from multiple threads, especially during the `readConfig` phase, will result in a corrupted state and unpredictable behavior. The asset loading system must guarantee that each builder instance is processed in isolation.

## API Surface
The public API is designed for internal use by the NPC asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns the final, immutable ActionDropItem. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionDropItem | O(N) | Deserializes the provided JSON, populating the builder's internal state. Throws exceptions on malformed data. |
| validate(...) | boolean | O(1) | Performs post-deserialization validation, such as checking if a throw is physically possible. Called by the asset loader. |
| getItem(BuilderSupport) | String | O(1) | Retrieves the configured item asset string. Intended for use by the resulting Action object. |
| getDropList(BuilderSupport) | String | O(1) | Retrieves the configured item drop list asset string. Intended for use by the resulting Action object. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the NPC asset loading framework. The conceptual flow is as follows.

```java
// Conceptual example of the asset loader's logic
JsonElement actionJson = parseNpcFile(".../behavior.json");

// The framework instantiates the correct builder for the action type
BuilderActionDropItem builder = new BuilderActionDropItem();

// The builder reads the config and is then used to create the final action
builder.readConfig(actionJson);
// ... validation steps ...
Action runtimeAction = builder.build(builderSupport);

// The 'builder' instance is now discarded.
// The 'runtimeAction' is added to the NPC's behavior tree.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BuilderActionDropItem()` in game logic. The NPC system manages the creation and lifecycle of builders during asset loading.
-   **Instance Re-use:** Do not attempt to cache and re-use a builder instance to parse multiple JSON objects. Each builder is stateful and designed for a single, one-shot configuration pass.
-   **Runtime Modification:** Do not hold a reference to a builder and attempt to modify its state after the `build` method has been called. The resulting Action will not reflect any subsequent changes.

## Data Pipeline
The BuilderActionDropItem serves as a critical transformation and validation step in the data pipeline that converts static configuration into live NPC behavior.

> Flow:
> NPC Behavior JSON File -> Server JSON Parser -> **BuilderActionDropItem** (`readConfig`, `validate`) -> `build()` -> ActionDropItem Instance -> NPC Behavior Tree -> Game Loop Execution

