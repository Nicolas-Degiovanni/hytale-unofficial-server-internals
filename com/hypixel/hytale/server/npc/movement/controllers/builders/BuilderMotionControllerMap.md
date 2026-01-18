---
description: Architectural reference for BuilderMotionControllerMap
---

# BuilderMotionControllerMap

**Package:** com.hypixel.hytale.server.npc.movement.controllers.builders
**Type:** Transient Factory / Builder

## Definition
```java
// Signature
public class BuilderMotionControllerMap extends BuilderBase<Map<String, MotionController>> implements ISpawnable {
```

## Architecture & Concepts
The BuilderMotionControllerMap class is a specialized factory within the server's NPC asset pipeline. Its primary responsibility is to deserialize a JSON array of motion controller definitions into a fully realized, runtime-usable Map of MotionController instances.

This class functions as a *composite builder*. It does not contain the logic for any single motion controller; instead, it orchestrates the creation of multiple, distinct MotionController objects by delegating to other, more specific builders. It holds a collection of these sub-builders, which are populated during the asset loading phase.

Crucially, this class also integrates with the server's spawning system by implementing the ISpawnable interface. This allows it to act as a gatekeeper during NPC spawn attempts. It aggregates the spawn eligibility of each individual motion controller it manages, ensuring that an NPC can only spawn if all its required motion controllers are valid for the given SpawningContext.

It is a critical bridge between static data configuration (JSON files) and dynamic server behavior (NPC movement and spawning).

### Lifecycle & Ownership
-   **Creation:** An instance of BuilderMotionControllerMap is created by the server's central BuilderManager during the asset loading phase. It is instantiated when the asset parser encounters a property that is defined to be a map of motion controllers. The `readConfig` method is immediately invoked to populate the builder from the corresponding JSON data.
-   **Scope:** The object's lifetime is tied to the asset definition it represents. It persists in memory as part of a parsed, but not yet fully built, NPC template. It is not a session-scoped or global object.
-   **Destruction:** The builder instance is eligible for garbage collection after the final NPC prototype has been constructed via the `build` method and cached by the NPC system. The resulting Map of MotionController objects is what persists, not the builder itself.

## Internal State & Concurrency
-   **State:** The core state is a private `BuilderObjectMapHelper` which manages a list of un-built MotionController builders. This state is highly mutable during the initial call to `readConfig`. After parsing is complete, the internal state should be considered effectively immutable. The public `build` method returns a *new* HashMap, preventing external mutation of the internal collection.
-   **Thread Safety:** This class is **not thread-safe** and must not be considered as such. It is designed to be operated exclusively by the single-threaded asset loading system. Concurrent calls to `readConfig` or `build` will result in undefined behavior, data corruption, and server instability. All interactions must be synchronized externally if multi-threaded access is unavoidable, though this is strongly discouraged.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Map | O(N) | Constructs the final Map of MotionController instances from the internal list of builders. N is the number of controllers. |
| readConfig(JsonElement) | Builder | O(N) | Deserializes a JSON array, populating the internal list of sub-builders. Throws exceptions on malformed data. |
| canSpawn(SpawningContext) | SpawnTestResult | O(N) | Implements ISpawnable. Iterates through all contained controllers, returning FAIL if any single controller cannot spawn. |
| validate(...) | boolean | O(N) | Recursively validates the configuration of this builder and all its contained motion controller builders. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic developers. It is an internal component of the asset pipeline, invoked automatically by the BuilderManager. A developer's interaction is purely declarative, through the definition of an NPC's JSON asset file.

The system internally uses the builder as follows:

```java
// Conceptual example of internal system usage
// 1. The asset loader creates the builder and populates it from JSON.
BuilderMotionControllerMap motionMapBuilder = builderManager.createBuilder(BuilderMotionControllerMap.class);
motionMapBuilder.readConfig(npcJson.get("motionControllers"));

// 2. Later, when building the final NPC prototype, the map is constructed.
Map<String, MotionController> runtimeControllers = motionMapBuilder.build(builderSupport);
npcPrototype.setMotionControllers(runtimeControllers);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BuilderMotionControllerMap()`. The object is uninitialized and will fail catastrophically if used. It must be created and managed by the server's asset loading framework.
-   **Calling build Before readConfig:** Invoking `build` on a builder that has not been populated via `readConfig` will produce an empty map, leading to NPCs with no movement logic.
-   **State Re-use:** Do not attempt to call `readConfig` multiple times on the same instance. The internal state is not designed for clearing or re-population.

## Data Pipeline

The BuilderMotionControllerMap serves two distinct data flows: asset construction and spawn validation.

**Asset Construction Flow:**
> Flow:
> NPC JSON Asset File -> Asset Loader -> **BuilderMotionControllerMap.readConfig** -> (Internal list of sub-builders is populated) -> **BuilderMotionControllerMap.build** -> Final `Map<String, MotionController>` -> Attached to NPC Prototype

**Spawn Validation Flow:**
> Flow:
> Server requests NPC spawn -> SpawningService -> NPC Prototype -> **BuilderMotionControllerMap.canSpawn** -> (Delegates to each MotionController's `canSpawn` method) -> Aggregated SpawnTestResult -> Spawn Allowed / Denied

