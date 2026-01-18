---
description: Architectural reference for BuilderEntityFilterNPCGroup
---

# BuilderEntityFilterNPCGroup

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterNPCGroup extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterNPCGroup is a key component within the server's data-driven NPC asset system. It functions as a transient factory, responsible for parsing a specific JSON configuration and constructing an immutable IEntityFilter instance, namely the EntityFilterNPCGroup.

This class embodies the **Builder Pattern**. Its primary role is to decouple the complex process of asset configuration from the final, runtime representation of an entity filter. During the server's asset loading phase, when an NPC behavior file is parsed, this builder is instantiated to handle the "filter by NPC group" logic. It reads string-based group names from a JSON structure, validates them, and prepares them for conversion into a more efficient, integer-based representation used by the game engine at runtime.

This class exists exclusively within the asset pipeline and is never referenced directly during the main game loop. It is a foundational piece for creating dynamic and configurable NPC behaviors without requiring direct code changes.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the asset loading framework when it encounters a JSON object corresponding to this filter type. The `readConfig` method is the primary entry point for its initialization.
- **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single JSON filter definition and building the corresponding runtime object.
- **Destruction:** The builder instance becomes eligible for garbage collection immediately after the `build` method is called and the resulting IEntityFilter is passed to the parent asset. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** The class is highly **mutable** during its configuration phase. The `readConfig` method populates the internal AssetArrayHolder fields, `includeGroups` and `excludeGroups`. This state is transient and serves only as an intermediate representation before the final filter object is built.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The asset loading pipeline is expected to process configurations in a single-threaded manner, or at a minimum, to provide a new builder instance for each configuration block. No synchronization mechanisms are present.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(N) | Constructs the final EntityFilterNPCGroup instance from the parsed state. N is the total number of groups. |
| readConfig(JsonElement) | Builder | O(N) | Parses the input JSON, populating the internal include and exclude group lists. This is the primary configuration method. |
| getIncludeGroups(BuilderSupport) | int[] | O(N) | Resolves the included NPC group names into their runtime integer ID representations. |
| getExcludeGroups(BuilderSupport) | int[] | O(N) | Resolves the excluded NPC group names into their runtime integer ID representations. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component, indicating if it is safe for production use. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is invoked internally by the asset system. The conceptual flow is as follows.

```java
// Conceptual example of how the asset system uses this builder
JsonElement filterConfig = parseNpcBehaviorFile("my_behavior.json");
BuilderEntityFilterNPCGroup builder = new BuilderEntityFilterNPCGroup();

// The asset system configures the builder from the JSON data
builder.readConfig(filterConfig);

// The builder creates the final, immutable filter object
// The builder instance is now discarded
IEntityFilter runtimeFilter = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to reuse a single builder instance to create multiple, different filter objects. Each builder is designed for a one-shot build operation.
- **Manual Population:** Do not manually populate the `includeGroups` or `excludeGroups` fields. Always use the `readConfig` method to ensure proper validation and data handling.
- **Usage Outside Asset Loading:** This class has dependencies on the asset loading context (BuilderSupport) and will fail if used in a standard game loop or other server subsystem.

## Data Pipeline
The builder acts as a transformation step in the data pipeline that converts declarative JSON configuration into executable server logic.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderEntityFilterNPCGroup.readConfig()** -> Internal AssetArrayHolder State -> **BuilderEntityFilterNPCGroup.build()** -> EntityFilterNPCGroup Instance -> Stored in Parent NPC Asset

---

