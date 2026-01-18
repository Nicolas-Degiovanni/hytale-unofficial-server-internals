---
description: Architectural reference for BuilderActionStartObjective
---

# BuilderActionStartObjective

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionStartObjective extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionStartObjective class is a server-side component within the NPC Behavior Asset Pipeline. It functions as a specialized factory and deserializer, responsible for translating a JSON configuration into a concrete, executable game action.

Its primary role is to parse a definition for a "start objective" action from an NPC asset file. During the server's asset loading phase, this builder validates the provided objective reference and constructs an immutable ActionStartObjective instance. This runtime instance is then integrated into the NPC's behavior tree, ready to be executed during a player interaction.

This class decouples the static asset definition (JSON) from the dynamic runtime logic (the ActionStartObjective object). It employs a two-phase loading pattern:
1.  **Configuration Phase:** The readConfig method consumes a JSON fragment, populating an internal AssetHolder with a reference to an objective. It performs initial validation, such as ensuring the referenced objective exists.
2.  **Build Phase:** The build method is called to produce the final runtime object, using a BuilderSupport context to resolve any dependencies or contextual information required by the final ActionStartObjective.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionStartObjective is created by the server's core asset loading system whenever it encounters the corresponding action type within an NPC behavior JSON file. Each definition in the JSON results in a new, separate instance of this builder.
-   **Scope:** The object's lifetime is extremely short and confined to the asset parsing and compilation process. It exists only to transform its corresponding JSON block into a runtime ActionStartObjective object.
-   **Destruction:** The builder instance is dereferenced and becomes eligible for garbage collection immediately after the call to its build method completes and the resulting action has been stored in the parent NPC behavior asset. It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** This class is **Mutable**. Its primary internal state is the objectiveId field, which is an AssetHolder. This field is populated by the readConfig method. The state is transient and exists only to bridge the gap between parsing the configuration and building the final action object.
-   **Thread Safety:** This class is **Not Thread-Safe** and must not be accessed concurrently. It is designed to be instantiated and used by a single thread within the server's asset loading pipeline. The sequence of calls (readConfig then build) is state-dependent and would be corrupted by concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionStartObjective | O(1) | Constructs the final, immutable runtime action object. |
| readConfig(JsonElement) | BuilderActionStartObjective | O(N) | Deserializes the JSON configuration, validates asset references, and populates internal state. Throws exceptions on invalid data. |
| getObjectiveId(BuilderSupport) | String | O(1) | Resolves and returns the concrete string identifier for the configured objective. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked exclusively by the server's asset management and NPC systems. The conceptual flow within that system is as follows.

```java
// Conceptual example of how the asset pipeline uses this builder
BuilderActionStartObjective builder = new BuilderActionStartObjective();

// 1. Configure the builder from a JSON source
JsonElement actionJson = parseActionFromJsonFile();
builder.readConfig(actionJson);

// 2. Build the runtime action using the server's context
BuilderSupport supportContext = server.getAssetBuildContext();
ActionStartObjective runtimeAction = builder.build(supportContext);

// 3. Integrate the action into the NPC's behavior tree
npcBehavior.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class in game logic. The NPC asset pipeline is solely responsible for its lifecycle. Manually creating one will result in an unconfigured and non-functional object.
-   **Reusing Instances:** Do not attempt to reuse a builder instance by calling readConfig multiple times. Each builder is designed for a single, linear lifecycle: new -> readConfig -> build -> discard.
-   **State Manipulation:** Do not access or modify the internal objectiveId field directly. Rely only on the public API contract.

## Data Pipeline
The BuilderActionStartObjective serves as a critical transformation step in the data pipeline that converts static NPC definitions into live server behavior.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionStartObjective.readConfig()** -> **BuilderActionStartObjective.build()** -> ActionStartObjective (Runtime Object) -> NPC Behavior Tree -> Player Interaction Event -> Action Execution -> Objective Started for Player

