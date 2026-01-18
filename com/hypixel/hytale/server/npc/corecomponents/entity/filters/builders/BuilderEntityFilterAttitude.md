---
description: Architectural reference for BuilderEntityFilterAttitude
---

# BuilderEntityFilterAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterAttitude extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterAttitude class is a factory component within the server's NPC asset pipeline. Its sole responsibility is to parse a JSON configuration snippet and construct an instance of EntityFilterAttitude. This class acts as a deserializer and validator, translating a game designer's declarative JSON definition into a concrete, executable IEntityFilter object used by the NPC's AI at runtime.

It is part of a larger, extensible system of Builders, where each builder is responsible for a specific type of component. This specific builder handles filters that test an NPC's disposition (e.g., *Hostile*, *Friendly*, *Neutral*) towards a potential target. The use of the Builder pattern decouples the complex asset loading and validation logic from the final, lightweight runtime filter object.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's asset loading framework when it encounters a corresponding filter type in an NPC's JSON definition file. It is never created directly by game logic code.
- **Scope:** This object is ephemeral. Its lifetime is strictly limited to the parsing and construction phase of a single IEntityFilter instance.
- **Destruction:** The builder is eligible for garbage collection immediately after the `build` method is called and the resulting EntityFilterAttitude is returned to the asset loader. It holds no persistent state beyond this process.

## Internal State & Concurrency
- **State:** The internal state, primarily the `attitudes` field, is mutable. The `readConfig` method populates this state from the input JSON. Once configuration is read, the state is effectively immutable for the remainder of its short lifecycle.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within the single-threaded context of the asset loading system. Concurrent calls to `readConfig` would result in a corrupted internal state and unpredictable behavior. The asset pipeline guarantees serialized access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterAttitude instance. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the JSON data, populates internal state. N is the number of attitudes. |
| getAttitudes(BuilderSupport) | EnumSet<Attitude> | O(1) | Retrieves the configured set of attitudes. Intended for use by the object it builds. |

## Integration Patterns

### Standard Usage
This builder is invoked automatically by the asset system. A developer or designer would define its use in a JSON file, not in Java code. The framework handles the lifecycle.

*Conceptual Framework Interaction:*
```java
// This code is conceptual and resides within the asset loading system.
// Do not replicate this pattern in game logic.

BuilderEntityFilterAttitude builder = new BuilderEntityFilterAttitude();
JsonElement filterConfig = getFilterJsonFromNpcDefinition();

builder.readConfig(filterConfig);
IEntityFilter runtimeFilter = builder.build(builderSupport);

// The 'runtimeFilter' is now attached to an NPC component.
// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually instantiate this class with `new BuilderEntityFilterAttitude()`. The asset system is responsible for managing builder lifecycles. Manual creation bypasses the framework's caching and validation layers.
- **State Re-use:** Do not attempt to reuse a builder instance to configure multiple filters. Each builder is single-use and its internal state is not designed to be reset.
- **Runtime Invocation:** This is a configuration-time class. It must not be used or referenced in performance-critical game loop code.

## Data Pipeline
The flow represents the transformation of a designer's intent into a runtime object.

> Flow:
> NPC Definition (JSON file) -> Asset Loader -> **BuilderEntityFilterAttitude** -> EntityFilterAttitude (runtime instance) -> NPC Targeting Sensor

