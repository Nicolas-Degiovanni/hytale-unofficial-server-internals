---
description: Architectural reference for BuilderBodyMotionWanderInCircle
---

# BuilderBodyMotionWanderInCircle

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionWanderInCircle extends BuilderBodyMotionWanderBase {
```

## Architecture & Concepts
The BuilderBodyMotionWanderInCircle class is a key component of the server's data-driven NPC asset system. It functions as a transient, stateful builder responsible for interpreting a specific JSON configuration block and translating it into a concrete runtime object, BodyMotionWanderInCircle.

This class embodies the Builder pattern to decouple the complex process of asset definition (from JSON files) from the instantiation of live game components. Its primary role is to parse, validate, and temporarily store configuration parameters for an NPC's "wander in a circle" movement behavior. The final, immutable runtime component is produced only when the build method is invoked, ensuring that the game engine works with fully-configured, valid objects.

This pattern is critical for engine stability and designer workflow, allowing NPC behaviors to be defined entirely in data files without requiring code changes.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading pipeline when it encounters a movement component of this specific type within an NPC's JSON definition. It is never created manually by game logic.
- **Scope:** Its lifecycle is extremely short. It exists only for the duration of parsing a single NPC asset.
- **Destruction:** The builder instance is discarded and becomes eligible for garbage collection immediately after its build method is called and the resulting BodyMotionWanderInCircle component is attached to the parent NPC definition. It holds no persistent state or engine resources.

## Internal State & Concurrency
- **State:** This object is highly mutable by design. Its core purpose is to accumulate state from the readConfig method. The internal fields, such as radius, flock, and useSphere, are populated during the parsing phase. This state is temporary and specific to the single component instance it is configured to build.

- **Thread Safety:** This class is **not thread-safe** and must only be used within a single-threaded context. The asset loading system guarantees this by processing one asset at a time in a sequential manner. Concurrent calls to readConfig or build would result in a corrupted or unpredictable configuration state.

## API Surface
The public API is designed for a single, linear workflow: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderBodyMotionWanderInCircle | O(1) | Parses the provided JSON, populating internal fields. Validates inputs and applies defaults. |
| build(BuilderSupport) | BodyMotionWanderInCircle | O(1) | Constructs and returns the final, immutable runtime component using the accumulated state. |
| getShortDescription() | String | O(1) | Provides metadata for tooling and editors. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Provides metadata on the stability of this component for tooling. |

## Integration Patterns

### Standard Usage
This class is exclusively used by the server's asset loading framework. A developer would never interact with it directly. The framework orchestrates its lifecycle.

```java
// Conceptual example of framework usage
JsonElement movementConfig = npcJson.get("movement");
String movementType = movementConfig.get("type").getAsString(); // e.g., "WanderInCircle"

// Framework resolves "WanderInCircle" to the BuilderBodyMotionWanderInCircle class
BuilderBodyMotionWanderInCircle builder = new BuilderBodyMotionWanderInCircle();

// The builder is configured from the data file
builder.readConfig(movementConfig);

// The final, runtime-ready component is created
BuilderSupport support = ...; // The context provided by the asset system
BodyMotionWanderInCircle runtimeComponent = builder.build(support);

// The component is attached to the NPC
npc.addMovementComponent(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually instantiate this class with `new`. The asset system is responsible for its creation based on data file definitions.
- **State Re-use:** Do not attempt to cache or re-use a builder instance. Each component must be built from a fresh builder to prevent state leakage and ensure configuration correctness.
- **Calling Build Prematurely:** Invoking build before readConfig will result in a component configured entirely with default values, which will almost certainly lead to incorrect NPC behavior.

## Data Pipeline
The flow of configuration data through this component is linear and unidirectional, transforming declarative data into an executable object.

> Flow:
> NPC JSON Asset File -> Asset Parser -> **BuilderBodyMotionWanderInCircle.readConfig()** -> Internal State Population -> **BuilderBodyMotionWanderInCircle.build()** -> BodyMotionWanderInCircle Instance -> NPC Entity Component List

