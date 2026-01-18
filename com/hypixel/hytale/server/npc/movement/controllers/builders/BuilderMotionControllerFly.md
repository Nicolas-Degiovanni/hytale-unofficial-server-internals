---
description: Architectural reference for BuilderMotionControllerFly
---

# BuilderMotionControllerFly

**Package:** com.hypixel.hytale.server.npc.movement.controllers.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderMotionControllerFly extends BuilderMotionControllerBase {
```

## Architecture & Concepts
The BuilderMotionControllerFly class is a concrete implementation of the Builder pattern, responsible for constructing a fully configured MotionControllerFly instance from a declarative data source, typically a JSON asset file.

It serves as a critical component within the server's NPC asset loading pipeline. Its primary role is to act as a data-to-object bridge, translating high-level configuration parameters for flying NPCs into a live, operational motion controller. The system identifies this specific builder using the string key "fly", returned by the getType method, allowing a central factory to delegate the construction process for any NPC defined with flight capabilities.

This class encapsulates all logic related to parsing, validating, and defaulting the numerous parameters that define an NPC's flight physics, such as speed, acceleration, and turning rates. By isolating this configuration logic, the core MotionControllerFly class remains clean, focused solely on runtime physics calculations.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system. A higher-level factory or manager reads an NPC definition, identifies a motion controller of type "fly", and creates a BuilderMotionControllerFly instance to process the corresponding JSON data block.
- **Scope:** Ephemeral and short-lived. An instance of this builder exists only for the duration of parsing and building a single MotionControllerFly. It is a temporary, single-use object.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and its product, the MotionControllerFly instance, has been returned to the asset system. It holds no persistent references and is not intended to outlive the asset loading transaction.

## Internal State & Concurrency
- **State:** Highly mutable. The object's internal fields (e.g., minAirSpeed, maxClimbSpeed) are populated sequentially as the `readConfig` method processes a JSON element. The builder's state represents the configuration of the object *under construction*.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context within the asset loading pipeline. Concurrent calls to `readConfig` on the same instance will lead to data corruption and unpredictable flight behavior in the resulting controller. The asset loading framework must enforce serialized access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | MotionControllerFly | O(1) | Constructs and returns the final MotionControllerFly using the internal state populated by `readConfig`. |
| readConfig(JsonElement) | BuilderMotionControllerFly | O(N) | Parses the provided JSON, populates the builder's internal fields, and applies validators. N is the number of keys. Returns `this` for chaining. |
| canSpawn(SpawningContext) | SpawnTestResult | O(1) | Executes pre-spawn validation logic to determine if an NPC with this controller can spawn in the given world context. |
| getType() | String | O(1) | Returns the unique string identifier "fly", used for factory registration and lookup. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the NPC asset system. The typical flow is managed by a central factory.

```java
// Pseudo-code for the asset loading system
JsonElement motionControllerJson = npcAsset.get("motionController");
String type = motionControllerJson.get("type").getAsString(); // "fly"

// Factory finds the correct builder based on type
BuilderMotionControllerBase builder = MotionControllerFactory.getBuilder(type);

// The factory configures and builds the final object
builder.readConfig(motionControllerJson);
MotionController controller = builder.build(builderSupport);

// The controller is then attached to the NPC
npc.setMotionController(controller);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderMotionControllerFly()`. The asset loading system is responsible for managing the lifecycle of builders. Manually creating one bypasses the registration and factory mechanisms.
- **State Re-use:** Do not call `build` multiple times on the same builder instance to create multiple controllers. The builder's state is not guaranteed to be clean after the first build and is not designed for reuse.
- **Configuration After Build:** Do not call `readConfig` or modify the builder after `build` has been invoked. The constructed MotionControllerFly will not reflect any subsequent changes.

## Data Pipeline
The builder acts as a transformation stage in the NPC creation pipeline, converting declarative data into an executable object.

> Flow:
> NPC Definition (JSON File) -> Server Asset Parser -> Motion Controller Factory -> **BuilderMotionControllerFly.readConfig()** -> **BuilderMotionControllerFly.build()** -> MotionControllerFly Instance -> Attached to NPC Entity State

