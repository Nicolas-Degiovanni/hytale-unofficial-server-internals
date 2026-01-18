---
description: Architectural reference for BuilderActionResetSearchRays
---

# BuilderActionResetSearchRays

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionResetSearchRays extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionResetSearchRays class is a factory component within the server-side NPC asset system. Its primary role is to deserialize a JSON configuration snippet and construct a runtime-executable `ActionResetSearchRays` object. This class acts as a bridge between the declarative NPC behavior definitions stored in asset files and the imperative action objects that are executed by the game engine.

In the Hytale NPC AI framework, "Search Rays" are sensors used by an NPC to probe its environment, detecting terrain, entities, or other game objects. The results of these probes are often cached for performance. This builder creates an action that explicitly invalidates and clears these cached results, forcing the NPC to perform fresh scans.

This pattern of using dedicated builder classes for each action type allows the system to be highly extensible. Adding a new NPC action requires creating a corresponding builder and runtime action class, without modifying the core asset loading machinery.

### Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionResetSearchRays is created by the NPC asset loading pipeline when it encounters the corresponding action type (e.g., "reset_search_rays") within an NPC's behavior definition JSON file.
-   **Scope:** The object's lifetime is extremely short. It exists only for the duration of parsing a single JSON action block.
-   **Destruction:** After the `build` method is called and the resulting `ActionResetSearchRays` object is integrated into the NPC's behavior tree, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** This class holds mutable state in the `names` field, which is an AssetArrayHolder. This field stores the string identifiers of the search ray sensors to be reset, as read from the JSON configuration. The state is transient and only serves as an intermediate representation before the final `Action` is built.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and used within a single-threaded asset loading context. The mutable internal state and the expectation of a sequential `readConfig` -> `build` lifecycle make it unsuitable for concurrent access. The asset loading system must provide the necessary synchronization guarantees.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(N) | Constructs the final `ActionResetSearchRays` instance. Complexity is proportional to the number of ray names to resolve. |
| readConfig(JsonElement) | BuilderActionResetSearchRays | O(N) | Parses the JSON data, populating the internal `names` holder. Complexity is proportional to the number of names in the JSON array. |
| getIds(BuilderSupport) | int[] | O(N) | Translates the configured string names into their corresponding integer slot IDs using the provided `BuilderSupport` context. This is a critical translation step. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked exclusively by the server's NPC asset loading system. The typical internal flow is as follows.

```java
// Hypothetical usage within the asset loading system
BuilderActionResetSearchRays builder = new BuilderActionResetSearchRays();

// 1. Configure the builder from the JSON asset
JsonElement configData = parseJson("{\"Names\": [\"forward_ground_check\", \"obstacle_check\"]}");
builder.readConfig(configData);

// 2. Build the runtime action using the current context
Action runtimeAction = builder.build(assetLoadingSupport);

// 3. The builder is now discarded
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class in game logic code. NPC behaviors should be defined entirely within JSON assets.
-   **State Reuse:** Do not attempt to reuse a builder instance to configure a second action. Each JSON action block must be processed by a new builder instance to prevent state corruption.
-   **Calling `build` before `readConfig`:** Invoking `build` on a non-configured builder will produce an action that does nothing, as the list of search ray names will be empty.

## Data Pipeline
The primary function of this class is to transform declarative data from a text format into an executable object within the game engine's memory.

> Flow:
> NPC Behavior JSON Asset -> Asset Deserializer -> **BuilderActionResetSearchRays.readConfig()** -> **BuilderActionResetSearchRays.build()** -> `ActionResetSearchRays` Instance -> NPC Behavior Tree -> Game Loop Execution

