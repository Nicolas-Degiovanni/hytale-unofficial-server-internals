---
description: Architectural reference for BuilderBodyMotionWander
---

# BuilderBodyMotionWander

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionWander extends BuilderBodyMotionWanderBase {
```

## Architecture & Concepts
The BuilderBodyMotionWander class is a factory component within the server-side NPC asset pipeline. Its sole responsibility is to translate a declarative JSON configuration into a concrete, runtime instance of the BodyMotionWander behavior component.

This class embodies the Builder pattern, providing a clean separation between the complex process of NPC definition (data) and the creation of live, in-game behavior objects. It is discovered and utilized by the asset loading system, which dynamically instantiates builders based on component types specified in an NPC's JSON file. This architecture allows for a highly extensible system where new movement behaviors can be added without modifying the core asset loading logic; one only needs to provide a new builder implementation.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading service when it parses a JSON configuration block corresponding to the "BodyMotionWander" component.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of a single BodyMotionWander object's construction.
- **Destruction:** The builder becomes eligible for garbage collection immediately after its `build` method is invoked and the resulting BodyMotionWander object is attached to the NPC being constructed. It holds no persistent references and is not managed by any container.

## Internal State & Concurrency
- **State:** Mutable. The primary state of this object consists of the configuration parameters for the wander behavior (e.g., wander radius, frequency, speed). This state is populated by the `readConfig` method, which deserializes data from a JSON source. The state is transient and only used to initialize the final BodyMotionWander object.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used exclusively within the single-threaded context of an NPC asset loading sequence. Concurrent access would lead to a corrupted or unpredictable configuration state.

## API Surface
The public API is minimal, focusing entirely on the build process and providing metadata for development tools.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionWander | O(1) | Constructs and returns a new BodyMotionWander instance using the internal configuration state. |
| readConfig(JsonElement) | BuilderBodyMotionWanderBase | O(N) | Populates the builder's internal state from a JSON object. N is the number of properties in the JSON. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable name for the component, used in tooling or logs. |
| getLongDescription() | String | O(1) | Provides a more detailed description of the component's function. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component's API, indicating if it is safe for production use. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the server's asset systems. The typical internal flow is as follows.

```java
// Pseudo-code representing the asset loader's logic
JsonElement wanderConfig = npcJson.get("motionComponent");
BuilderBodyMotionWander builder = new BuilderBodyMotionWander();

// 1. Configure the builder from the data file
builder.readConfig(wanderConfig);

// 2. Construct the final runtime object
BuilderSupport context = ...; // Get context from asset loader
BodyMotionWander wanderMotion = builder.build(context);

// 3. Attach the component to the NPC
npc.setMotionComponent(wanderMotion);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not hold a reference to a builder and reuse it to build a second BodyMotionWander instance. Each builder is stateful and intended for a single build operation. Reusing it will result in multiple NPCs sharing an improperly configured or identical behavior definition.
- **Skipping Configuration:** Calling `build` without a preceding call to `readConfig` will produce a BodyMotionWander component with default, uninitialized parameters. This will likely result in inert or unpredictable NPC movement.
- **Manual Instantiation:** Game logic should never instantiate this class directly. NPC behaviors should always be defined in data files and loaded through the asset pipeline to ensure consistency and data-driven design.

## Data Pipeline
The builder acts as a transformation step, converting serialized data into an executable game object.

> Flow:
> NPC Definition (*.json file*) -> Server Asset Loader -> **BuilderBodyMotionWander**.readConfig() -> **BuilderBodyMotionWander**.build() -> BodyMotionWander (runtime component) -> NPC Entity State

