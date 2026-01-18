---
description: Architectural reference for BuilderActionPlaceBlock
---

# BuilderActionPlaceBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionPlaceBlock extends BuilderActionBase {
```

## Architecture & Concepts

The BuilderActionPlaceBlock is a configuration-time factory component within the server's NPC Behavior system. Its sole responsibility is to translate a declarative JSON configuration block into a concrete, executable runtime object: the ActionPlaceBlock. It acts as a deserializer and constructor, bridging the gap between static NPC asset definitions and their live, in-world behavior.

This class is a fundamental part of the engine's data-driven design for AI. Instead of hard-coding NPC behaviors, developers define them in JSON files. The engine's asset loading pipeline discovers these definitions and uses corresponding builder classes, like this one, to construct the nodes of an NPC's behavior tree.

A key architectural pattern employed here is the use of Holder objects, such as DoubleHolder and BooleanHolder. These wrappers allow configuration values to be resolved dynamically at runtime via the ExecutionContext. This enables behaviors that can adapt to the NPC's current state or environment, rather than being defined by static constants.

Furthermore, the call to `requireFeature` establishes a contract. It declares that any NPC using this action *must* have the Position feature available in its runtime context, ensuring the necessary data is present for the action to execute successfully.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderActionPlaceBlock is created by the server's NPC asset loading system whenever it encounters the corresponding action type (e.g., "placeBlock") within an NPC's JSON behavior definition. It is never instantiated directly by game logic.

-   **Scope:** The builder's lifecycle is unexpectedly long and is a critical detail for memory management analysis. While its primary purpose is fulfilled during the initial asset load, the resulting ActionPlaceBlock instance maintains a direct reference to the builder that created it. This is because the live action calls back to the builder's accessor methods (`getRange`, `isAllowEmptyMaterials`) to retrieve its configured parameters. Therefore, the builder object persists in memory for the entire lifetime of the Action it produces, which is typically the lifetime of the NPC itself.

-   **Destruction:** The BuilderActionPlaceBlock instance is eligible for garbage collection only when the ActionPlaceBlock it created is destroyed. This typically occurs when an NPC is unloaded or its behavior tree is replaced.

## Internal State & Concurrency

-   **State:** The class is stateful and highly mutable during its configuration phase. The `readConfig` method populates the internal `range` and `allowEmptyMaterials` Holder fields from the source JSON. After the `build` method has been invoked, the builder's state should be considered effectively immutable. Any subsequent modification would lead to undefined behavior in the live Action.

-   **Thread Safety:** This class is **not thread-safe**. All instances are created, configured, and used within the server's single-threaded asset loading pipeline. It is a strict violation to access or modify a builder instance from multiple threads. The resulting Action may be executed on a different thread (e.g., the main server tick thread), but its access to the builder's state is read-only via the accessor methods.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionPlaceBlock instance. This is the primary factory method. |
| readConfig(JsonElement) | BuilderActionPlaceBlock | O(1) | Deserializes the JSON configuration into the builder's internal state. Must be called before `build`. |
| getRange(BuilderSupport) | double | O(1) | Runtime accessor for the configured range. Called by the live Action. |
| isAllowEmptyMaterials(BuilderSupport) | boolean | O(1) | Runtime accessor for the empty materials flag. Called by the live Action. |

## Integration Patterns

### Standard Usage

The builder is used exclusively by the internal asset loading system. The pattern is a direct sequence of instantiation, configuration, and building.

```java
// This logic resides deep within the NPC asset loading system.
// 1. A builder is instantiated based on a "type" field in JSON.
BuilderActionPlaceBlock builder = new BuilderActionPlaceBlock();

// 2. The relevant JSON object is passed to configure the builder.
builder.readConfig(jsonActionObject);

// 3. The configured builder is used to create the runtime Action.
Action runtimeAction = builder.build(builderSupport);

// 4. The action is added to the NPC's behavior tree.
behaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Game logic or plugin code must never instantiate a builder directly using `new`. Behavior tree construction is managed entirely by the server's asset systems.
-   **State Mutation After Build:** Modifying the builder's state after `build` has been called is a severe anti-pattern. The live `Action` reads from this state, and changing it post-build will cause unpredictable behavior changes in active NPCs.
-   **Builder Reuse:** A single builder instance is designed to configure and build a single `Action` instance. Do not attempt to reuse a builder to create multiple actions, as its internal state is not designed for this.

## Data Pipeline

The BuilderActionPlaceBlock functions as a critical step in the configuration-to-runtime data pipeline for NPC behaviors.

> Flow:
> NPC Definition (*.json file*) -> Server Asset Parser -> **BuilderActionPlaceBlock.readConfig()** -> **BuilderActionPlaceBlock.build()** -> ActionPlaceBlock Instance -> Live NPC Behavior Tree

