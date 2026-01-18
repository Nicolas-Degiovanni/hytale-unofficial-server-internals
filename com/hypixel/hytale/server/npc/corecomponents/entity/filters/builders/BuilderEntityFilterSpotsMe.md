---
description: Architectural reference for BuilderEntityFilterSpotsMe
---

# BuilderEntityFilterSpotsMe

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterSpotsMe extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterSpotsMe class is a key component of Hytale's data-driven NPC asset system. It embodies the Builder pattern, acting as a deserializer and factory for a specific type of entity filter: one that determines if an NPC can "spot" another entity based on line-of-sight and viewing angles.

Its primary role is to bridge the gap between static JSON configuration files and live, in-game Java objects. Developers define NPC sensory parameters in JSON, and during the server's asset loading phase, this builder is responsible for parsing that data and constructing a fully configured EntityFilterSpotsMe instance.

This class is not intended for direct use in game logic. Instead, it operates exclusively within the asset pipeline, translating declarative configuration into executable behavior. It handles important data transformations, such as converting angles from degrees (a human-friendly format in JSON) to radians (the format required by the engine's math libraries).

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's asset loading framework, specifically the BuilderSupport system. When the loader parses an NPC definition and encounters a filter component of this type, it creates a new BuilderEntityFilterSpotsMe instance to handle the configuration.
- **Scope:** The object's lifetime is extremely short and confined to the asset build process. It exists only to parse a single JSON block and produce one EntityFilterSpotsMe object.
- **Destruction:** After the build method is called and the resulting filter is passed back to the asset loader, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency
- **State:** This class is fundamentally mutable. Its fields—viewAngle, testLineOfSight, and viewTest—are populated sequentially as the readConfig method processes a JSON element. The state is transient and serves as a temporary container for parameters before the final, immutable filter object is constructed.
- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed for synchronous, single-threaded execution within the asset loading pipeline. Concurrent calls to readConfig or build would result in a race condition and produce a corrupted or unpredictable filter object.

## API Surface
The public API is designed for use by the asset building system, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | EntityFilterSpotsMe | O(1) | Constructs and returns the final EntityFilterSpotsMe object using the state accumulated from readConfig. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(1) | Parses the provided JSON data, populates internal fields, and performs data validation and transformation (e.g., degrees to radians). |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for use in tooling or editor environments. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this builder, indicating if it is ready for production use. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class via Java code. Instead, they define the filter's properties within an NPC's JSON asset file. The engine handles the rest.

```json
// Example NPC component definition in a .json file
{
  "component": "EntityFilterSpotsMe",
  "ViewAngle": 90.0,
  "ViewTest": "VIEW_SECTOR",
  "TestLineOfSight": true
}
```

The server's asset loader will identify the component, instantiate BuilderEntityFilterSpotsMe, and pass this JSON object to its readConfig method.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderEntityFilterSpotsMe()`. The asset system is responsible for the lifecycle of builders. Manually creating one bypasses the asset pipeline and will not be integrated into any NPC.
- **State Re-use:** Do not attempt to reuse a builder instance to build multiple filters. The internal state is not reset between builds and will lead to incorrect configurations. Each filter defined in JSON results in a new, single-use builder.
- **Calling build Before readConfig:** Invoking build on a fresh instance without first calling readConfig will produce a filter with default values, ignoring any intended JSON configuration and likely causing unintended NPC behavior.

## Data Pipeline
This builder acts as a transformation step within the larger NPC asset loading pipeline.

> Flow:
> NPC JSON Asset File -> Asset Loader/Parser -> **BuilderEntityFilterSpotsMe** (deserializes JSON) -> EntityFilterSpotsMe Instance -> NPC Sensor Component -> Real-time Game Evaluation

