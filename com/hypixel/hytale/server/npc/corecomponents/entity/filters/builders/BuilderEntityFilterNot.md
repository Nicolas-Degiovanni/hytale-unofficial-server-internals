---
description: Architectural reference for BuilderEntityFilterNot
---

# BuilderEntityFilterNot

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderEntityFilterNot extends BuilderEntityFilterWithToggle {
```

## Architecture & Concepts
The BuilderEntityFilterNot class is a component of the server's data-driven NPC configuration system. It functions as a concrete implementation of the Builder pattern, specifically designed to translate a JSON configuration into a runtime instance of an EntityFilterNot.

Its primary role is to enable logical inversion within entity filtering logic. In the Hytale engine, NPC behaviors, targeting, and other AI constructs rely on filters (IEntityFilter) to make decisions. This builder allows content creators and designers to define complex conditional logic directly in asset files without writing code. For example, a behavior can be configured to target any entity that is *not* a player.

This class is not used directly in gameplay logic. Instead, it is discovered and invoked by a central BuilderManager during the server's asset loading phase. It parses a specific JSON structure, constructs the nested filter it is configured to invert, and then wraps it within an EntityFilterNot object. This compositional approach allows for deeply nested and complex logical filter chains.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the asset pipeline's BuilderManager. This occurs when the system parses an NPC JSON configuration file and encounters a filter definition with a type corresponding to this builder. Direct instantiation by developers is an anti-pattern.
- **Scope:** Extremely short-lived. An instance of BuilderEntityFilterNot exists only for the duration of parsing and building a single filter definition from a configuration file.
- **Destruction:** The object becomes eligible for garbage collection immediately after its build method has been called and the resulting IEntityFilter has been returned to the BuilderManager. It does not persist in memory after asset loading is complete.

## Internal State & Concurrency
- **State:** The class is **mutable**. Its internal state, primarily the BuilderObjectReferenceHelper for the nested filter, is populated by the readConfig method. This state is transient and essential for the multi-step process of reading configuration and then building the final object.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The entire NPC asset building process is designed to be a single-threaded, synchronous operation. Concurrent calls to its methods would lead to race conditions and an unpredictable, corrupt final state.

## API Surface
The public API is designed for consumption by the automated builder system, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(N) | Constructs and returns the final EntityFilterNot instance. Complexity is dependent on the build complexity (N) of the nested filter. Returns null if the nested filter cannot be built. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(1) | Parses the provided JSON data to configure the builder's internal state, specifically identifying the nested filter to be inverted. |
| validate(...) | boolean | O(N) | Recursively validates the configuration for this builder and its nested filter. Used during server startup to detect invalid NPC configurations. |
| getFilter(BuilderSupport) | IEntityFilter | O(N) | A helper method to build and retrieve the nested filter instance that will be inverted. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this Java class directly. Instead, they define the filter logic declaratively within an NPC's JSON asset file. The engine's asset loading system handles the instantiation and invocation of this builder.

**Conceptual JSON Configuration:**
```json
{
  "component": "TargetSelector",
  "filter": {
    "type": "Not",
    "filter": {
      "type": "IsPlayer"
    }
  }
}
```
In this example, when the server loads the NPC, the BuilderManager will see the type "Not", instantiate a BuilderEntityFilterNot, and pass it the corresponding JSON object. The builder will then recursively process the inner "IsPlayer" filter, ultimately producing a runtime filter that matches any entity that is not a player.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderEntityFilterNot()`. The class is not designed for manual setup and relies on the context provided by the BuilderManager and BuilderSupport systems for its operation.
- **State Reuse:** Do not attempt to cache and reuse an instance of this builder. Each instance is stateful and intended for a single, one-shot build operation from a specific JSON configuration block.
- **Manual Configuration:** Do not call `readConfig` and `build` manually. This bypasses the validation, dependency resolution, and lifecycle management provided by the core asset system, which can lead to runtime errors or invalid game state.

## Data Pipeline
This builder acts as a transformation step in the server's asset loading pipeline, converting declarative data into executable game logic.

> Flow:
> NPC JSON File on Disk -> Asset Deserializer -> BuilderManager -> **BuilderEntityFilterNot** -> EntityFilterNot (Runtime Object) -> NPC Behavior Component

