---
description: Architectural reference for BuilderMotionControllerDive
---

# BuilderMotionControllerDive

**Package:** com.hypixel.hytale.server.npc.movement.controllers.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderMotionControllerDive extends BuilderMotionControllerBase {
```

## Architecture & Concepts
The BuilderMotionControllerDive class is a **configurable factory** within the server's NPC asset pipeline. It serves as the bridge between declarative JSON asset definitions and the concrete, in-game implementation of an NPC's diving and swimming physics, the MotionControllerDive.

Its core architectural purpose is to decouple the high-level design of an NPC's movement (defined by designers in JSON files) from the low-level physics simulation. It achieves this by consuming a JSON data structure, validating its parameters, and using them to construct a fully configured MotionControllerDive instance.

Beyond its role as a factory, this class also integrates with the server's spawning system. The `canSpawn` method acts as a predicate, allowing the engine to verify that an NPC requiring diving capabilities is only spawned in a valid environment (i.e., in water of sufficient depth). This prevents NPCs from spawning in invalid locations where their primary movement system would be non-functional.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system when it encounters a motion controller of type `dive` in an NPC's JSON definition. This process is automated and driven by asset metadata.
-   **Scope:** This object is **transient and short-lived**. Its existence is scoped to the asset loading and NPC instantiation process. It is created, configured via `readConfig`, used once to `build` a MotionControllerDive, and then becomes eligible for garbage collection.
-   **Destruction:** The builder holds no persistent state or external references. It is cleaned up by the garbage collector once the `build` method has returned and the calling scope (within the asset loader) is left.

## Internal State & Concurrency
-   **State:** The BuilderMotionControllerDive is highly **mutable**. Its internal fields (e.g., minHorizontalSpeed, gravity, maxDiveDepth) are populated and modified exclusively through the `readConfig` method. The state of the builder directly represents the configuration that will be passed to the final MotionControllerDive instance.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be used within the single-threaded context of the server's asset loading pipeline. Concurrent calls to `readConfig` would result in a corrupted and unpredictable configuration state. The engine must enforce a strict, single-threaded access pattern for this object.

## API Surface
The public API is designed for a sequential, single-use workflow by the asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderMotionControllerDive | O(N) | Parses the provided JSON, populating and validating all internal movement parameters. N is the number of keys in the JSON object. |
| build(BuilderSupport support) | MotionControllerDive | O(1) | Constructs and returns a new MotionControllerDive instance using the currently configured state. |
| canSpawn(SpawningContext context) | SpawnTestResult | O(1) | Predicate used by the spawning system to determine if an NPC can spawn at the given location, primarily checking for water. |
| category() | Class | O(1) | Returns the base class type (MotionController) for categorization by the asset system. |
| getClassType() | Class | O(1) | Returns the concrete class type (MotionControllerDive) that this builder produces. |

## Integration Patterns

### Standard Usage
This class is intended for internal engine use during asset processing. A system-level service retrieves the appropriate builder, configures it from a JSON asset, and uses it to construct the final motion controller component which is then attached to an NPC entity.

```java
// Engine-level code during NPC asset loading
BuilderMotionControllerDive builder = new BuilderMotionControllerDive();

// Configure the builder from a parsed JSON asset file
JsonElement motionConfig = npcAssetJson.get("motionController");
builder.readConfig(motionConfig);

// Construct the final controller instance
BuilderSupport support = ...; // Engine-provided context
MotionControllerDive diveController = builder.build(support);

// Attach the controller to the NPC
npc.setMotionController(diveController);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not attempt to cache and reuse a BuilderMotionControllerDive instance. Its internal state is mutated by `readConfig`, making it unsafe for building more than one unique controller configuration. A new builder must be instantiated for each asset definition.
-   **Calling Build Before Configuration:** Invoking `build` before `readConfig` will produce a MotionControllerDive with uninitialized or default-zero values for its physics parameters, leading to unpredictable or non-functional NPC movement.
-   **Manual Instantiation:** Game logic and modders should never instantiate this class directly. The creation of motion controllers should be driven entirely by the declarative NPC JSON assets.

## Data Pipeline
The BuilderMotionControllerDive is a critical transformation step in the data pipeline that turns a static asset file into a dynamic, in-world entity.

> Flow:
> NPC Asset JSON File -> Server Asset Parser -> **BuilderMotionControllerDive.readConfig()** -> In-Memory Parameter State -> **BuilderMotionControllerDive.build()** -> MotionControllerDive Instance -> Attached to NPC Entity State

