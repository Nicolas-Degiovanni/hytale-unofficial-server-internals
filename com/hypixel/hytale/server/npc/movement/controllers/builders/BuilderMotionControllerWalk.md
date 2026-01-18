---
description: Architectural reference for BuilderMotionControllerWalk
---

# BuilderMotionControllerWalk

**Package:** com.hypixel.hytale.server.npc.movement.controllers.builders
**Type:** Transient Factory / Builder

## Definition
```java
// Signature
public class BuilderMotionControllerWalk extends BuilderMotionControllerBase {
```

## Architecture & Concepts
The **BuilderMotionControllerWalk** is a configuration factory responsible for deserializing and validating NPC walking physics from a JSON asset definition. It serves as the critical bridge between declarative NPC behavior defined in JSON files and the concrete, runtime **MotionControllerWalk** object used by the server's physics and AI systems.

Its primary architectural function is to enforce a strict contract for NPC motion properties. It parses dozens of parameters—such as gravity, jump height, acceleration, and climb speed—providing default values for missing fields and executing validation logic to ensure the final configuration is physically coherent.

A key architectural feature is the use of holder classes like **DoubleHolder** and **FloatHolder**. This pattern decouples the configuration from a static value, allowing properties to be resolved dynamically at runtime via a **BuilderSupport** context. This enables advanced behaviors where an NPC's physical capabilities could change based on game state, difficulty, or other contextual factors without requiring a different asset file. This class effectively translates a static data asset into a potentially dynamic, state-aware motion profile.

## Lifecycle & Ownership
- **Creation:** An instance of **BuilderMotionControllerWalk** is created by the server's asset pipeline or NPC factory system whenever an NPC definition containing a *walk* motion controller is loaded. It is not intended for direct instantiation by game logic developers.

- **Scope:** The object is transient and has a very short lifecycle, scoped exclusively to the asset loading and instantiation process. It exists only to parse a single JSON configuration, validate it, and produce one **MotionControllerWalk** instance.

- **Destruction:** After the **build** method is called and the resulting **MotionControllerWalk** is returned, the builder instance holds no further purpose and is eligible for garbage collection. It maintains no persistent state and is not registered in any service locator.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. The primary entry point, **readConfig**, populates over 40 internal fields directly from the provided JSON data. The state is a temporary representation of the configuration before it is passed to the final **MotionControllerWalk** object.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within an asset loading context.

    **WARNING:** Accessing a shared instance from multiple threads will lead to race conditions and unpredictable, corrupt motion controller configurations. Each configuration parsing operation must use a new, dedicated builder instance.

## API Surface
The public API is designed for a sequential, single-pass configuration process. The most critical methods are the entry point (**readConfig**), the terminal operation (**build**), and the pre-spawn validation check (**canSpawn**). The numerous public getter methods exist solely to expose the configured state to the **MotionControllerWalk** constructor during the build phase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderMotionControllerWalk | O(N) | Parses the JSON object, populating the builder's internal state. N is the number of keys in the JSON. Returns itself for chaining. |
| build(BuilderSupport support) | MotionControllerWalk | O(1) | Constructs and returns the final, immutable MotionControllerWalk instance using the configured state. This is the terminal operation. |
| canSpawn(SpawningContext ctx) | SpawnTestResult | O(1) | Performs a predicate test to determine if an NPC with this motion profile can spawn at the given context, typically checking for solid ground. |

## Integration Patterns

### Standard Usage
The builder is used internally by the engine's NPC asset loading system. A developer's interaction is typically limited to defining the JSON file, not writing Java code that uses this builder. The conceptual engine-level usage is as follows.

```java
// Conceptual engine code for loading an NPC's motion controller
JsonElement motionConfig = loadNpcAssetFile("troll.json").get("motionController");
BuilderSupport support = createBuilderSupportForNpc(npc);

BuilderMotionControllerWalk builder = new BuilderMotionControllerWalk();
builder.readConfig(motionConfig);

MotionControllerWalk walkController = builder.build(support);
npc.setMotionController(walkController);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a builder instance to build a second controller. The internal state is not reset between calls to **build**, which will result in configuration bleed and undefined behavior.

- **Direct Instantiation:** Game logic code should never instantiate this builder directly. NPC motion controllers must be defined in assets and loaded through the proper engine systems to ensure correctness and compatibility.

- **Concurrent Modification:** Do not call **readConfig** on an instance from a different thread. The object is not designed for concurrent access and its state will become corrupted.

## Data Pipeline
The **BuilderMotionControllerWalk** is a specific stage in the larger NPC instantiation pipeline. It transforms structured text data into a live game component.

> Flow:
> NPC Definition (*.json file*) -> Asset Loader -> **BuilderMotionControllerWalk**.readConfig() -> **BuilderMotionControllerWalk**.build() -> **MotionControllerWalk** Instance -> NPC Entity -> Server Physics Engine

---

