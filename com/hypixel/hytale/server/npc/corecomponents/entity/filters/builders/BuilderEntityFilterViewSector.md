---
description: Architectural reference for BuilderEntityFilterViewSector
---

# BuilderEntityFilterViewSector

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderEntityFilterViewSector extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterViewSector is a component within the server-side NPC asset pipeline. Its primary function is to act as a deserializer and factory for a specific type of entity filter, the EntityFilterViewSector. It translates a declarative JSON configuration from an asset file into a live, executable filter object used by the NPC's AI.

This class embodies the Builder pattern, isolating the complex construction logic of an EntityFilterViewSector from the core asset loading system. It is responsible for:
1.  **Parsing:** Reading a JSON object to extract configuration values, such as the angle of the view sector.
2.  **Validation:** Ensuring the provided configuration values are within acceptable bounds (e.g., a view sector between 0 and 360 degrees) using validators like DoubleRangeValidator.
3.  **Context Assertion:** Declaring the required execution context for the filter it builds. The call to requireContext enforces a critical design constraint: this filter is only valid when used within a sensor that is evaluating *other* entities, not the NPC itself.
4.  **Instantiation:** Constructing the final, immutable EntityFilterViewSector instance, passing itself as a source of configuration data.

This builder is discovered and invoked by a higher-level asset management system, which maintains a registry of available builder types. It forms a bridge between static game data and the dynamic AI runtime.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the NPC asset loading framework when it encounters a corresponding filter type in a JSON definition file. It is never created directly via its constructor in game logic code.
-   **Scope:** The object's lifetime is extremely short and confined to the asset parsing process. It exists only to configure and build a single EntityFilterViewSector instance.
-   **Destruction:** Once the build method is called and the resulting IEntityFilter is returned to the asset loader, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the game state.

## Internal State & Concurrency
-   **State:** The builder maintains a mutable internal state, primarily the FloatHolder for the viewSector. This state is populated from the JSON configuration during the call to readConfig. This state is transient and discarded after the build operation.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and used within the context of a single-threaded asset loading operation. The internal state is mutable, and concurrent calls to readConfig or build would result in a corrupted or unpredictable filter configuration. The asset pipeline must guarantee that each builder instance is accessed by only one thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterViewSector instance. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the JSON data, validates inputs, and populates internal state. N is the number of keys in the JSON object. |
| getViewSectorRadians(BuilderSupport) | float | O(1) | Converts the configured view sector from degrees to radians. This is a deferred calculation, called by the created filter at runtime. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for use in development tools. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, e.g., Stable. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the asset system based on a JSON configuration. A game designer would define the filter in an NPC asset file.

**Example NPC Asset Snippet (JSON):**
```json
{
  "sensor": {
    "filters": [
      {
        "type": "EntityFilterViewSector",
        "config": {
          "ViewSector": 90.0
        }
      }
    ]
  }
}
```
The asset pipeline would identify the type "EntityFilterViewSector", instantiate this builder, and pass the contents of the config object to its readConfig method.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new BuilderEntityFilterViewSector(). The asset system is responsible for its lifecycle. Direct creation bypasses the registry and will not be integrated into the NPC.
-   **State Re-use:** Do not attempt to cache and re-use a builder instance. Each instance is designed to build one and only one filter object. Re-using it will lead to unpredictable behavior as its internal state is not reset.
-   **Manual Configuration:** Do not call methods like getFloat directly. The public contract for configuration is exclusively through the readConfig method.

## Data Pipeline
The builder is a key transformation step in the data pipeline that converts static asset definitions into runtime AI logic.

> Flow:
> NPC Definition JSON File -> Asset Loading Service -> Builder Registry -> **BuilderEntityFilterViewSector** (instantiated) -> EntityFilterViewSector (runtime object) -> NPC Entity Sensor -> AI Behavior Tree

---

