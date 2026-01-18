---
description: Architectural reference for BuilderBodyMotionPath
---

# BuilderBodyMotionPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionPath extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionPath class is a configuration-driven factory within the server-side NPC behavior system. It serves as a critical bridge between declarative asset definitions (JSON) and concrete, executable game logic. Its primary responsibility is to parse a JSON configuration block, validate its parameters, and construct an immutable BodyMotionPath runtime component.

This class embodies the Builder pattern. It is not a long-lived service but a transient object used during the NPC asset loading pipeline. The engine identifies the need for a pathing behavior from the NPC's JSON definition, instantiates this builder, populates it via the readConfig method, and then calls build to produce the final BodyMotionPath object. This resulting object is then attached to the NPC entity to control its movement.

This pattern decouples the data format (JSON) from the runtime implementation (BodyMotionPath), allowing designers to define complex NPC pathing behaviors without writing code.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's NPC asset loading system when it processes a behavior component of this type from an NPC's JSON definition file. It is never created directly by gameplay code.
- **Scope:** Extremely short-lived. An instance of BuilderBodyMotionPath exists only for the duration of parsing a single JSON object and building one BodyMotionPath component.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the build method returns. It holds no persistent state or external references and is designed to be discarded after use.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. Fields such as pathWidth, minRelativeSpeed, and direction are populated and modified during the call to readConfig. This state is transient and represents the complete configuration for a single BodyMotionPath instance. The use of Holder objects like EnumHolder and NumberArrayHolder allows for deferred value resolution based on an execution context, adding a layer of dynamic configuration.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the asset loading pipeline. Concurrent calls to readConfig or access to its internal fields will result in race conditions and a corrupted, unpredictable final object. This is a deliberate design choice to optimize for the typically sequential nature of asset processing.

## API Surface
The public API is focused on the build-and-configure lifecycle. Getters are provided primarily for internal access by the final BodyMotionPath object during its construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionPath | O(1) | Constructs and returns the final, configured BodyMotionPath runtime object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderBodyMotionPath | O(N) | Parses the input JSON, populates the builder's internal state, and performs validation. N is the number of properties in the JSON object. |
| getShape(BuilderSupport) | BodyMotionPath.Shape | O(1) | Resolves and returns the configured path shape, potentially using the provided execution context. |
| getDelayScaleRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured node pause scale range. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay developers. The following pseudocode illustrates its lifecycle as managed by the NPC asset loading system.

```java
// This logic resides within the server's asset loading framework.
// Do not replicate this pattern in gameplay code.

JsonElement behaviorConfig = parseNpcDefinitionFile("my_npc.json");
BuilderSupport support = createBuilderSupportForNpc();

BuilderBodyMotionPath builder = new BuilderBodyMotionPath();
builder.readConfig(behaviorConfig);

// The final, immutable runtime component is created here.
BodyMotionPath pathComponent = builder.build(support);

// The component is attached to the NPC entity. The builder is now discarded.
npcEntity.addMotionComponent(pathComponent);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class with `new BuilderBodyMotionPath()` in gameplay logic. NPC behaviors must be defined in JSON asset files to be correctly managed by the engine.
- **State Reuse:** Do not reuse a builder instance. A single builder must not be used to configure and build more than one BodyMotionPath component. Its internal state is not reset between build calls and will lead to incorrect or blended configurations.
- **Premature Build:** Calling build before readConfig has been successfully invoked will produce a component with default, uninitialized values, leading to unpredictable and incorrect NPC behavior.

## Data Pipeline
The BuilderBodyMotionPath is a key transformation step in the data pipeline that converts static asset definitions into live server objects.

> Flow:
> NPC Definition (*.json file*) -> JSON Parser -> `JsonElement` -> Asset Loading Service -> **BuilderBodyMotionPath.readConfig()** -> **BuilderBodyMotionPath.build()** -> `BodyMotionPath` (Runtime Component) -> Attached to NPC Entity

