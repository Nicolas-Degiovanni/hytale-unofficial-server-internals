---
description: Architectural reference for BuilderEntityFilterLineOfSight
---

# BuilderEntityFilterLineOfSight

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterLineOfSight extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterLineOfSight is a factory class within the server's NPC asset definition pipeline. Its sole responsibility is to interpret a configuration token (typically from a JSON file) and construct an instance of EntityFilterLineOfSight.

This class is a concrete implementation of the Builder pattern, designed to be discovered and managed by a higher-level asset loading service. It acts as a translator between declarative NPC configuration and executable in-game logic. The builder ensures that the resulting filter component is constructed correctly and validates its usage context via the `requireContext` method. This validation is a critical architectural safeguard, preventing designers from using this filter in illogical scenarios, such as an NPC checking for line of sight to itself.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system when it encounters the corresponding identifier in an NPC's JSON definition. It is never created manually by game logic developers.
- **Scope:** Extremely short-lived and transient. An instance exists only for the brief duration required to parse a configuration block and build the final IEntityFilter object.
- **Destruction:** The builder instance is immediately eligible for garbage collection after the `build` method returns its product. It holds no references and maintains no state beyond the build process.

## Internal State & Concurrency
- **State:** This class is stateless. It does not retain any information from the `readConfig` call as member variables. Its purpose is to perform a direct, one-shot transformation of configuration into an object.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. The entire NPC asset loading pipeline is expected to operate in a single-threaded context to ensure deterministic and safe component assembly. Accessing a builder instance from multiple threads will lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns a new EntityFilterLineOfSight instance. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(1) | Validates the context for this filter. Throws if used improperly. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component, indicating it is production-ready. |

## Integration Patterns

### Standard Usage
This class is not invoked directly via Java code. Instead, it is triggered declaratively by the asset system. A game designer would use it in an NPC's JSON definition file.

**Example NPC JSON Snippet:**
```json
{
  "name": "LookoutGoblin",
  "ai": {
    "goals": [
      {
        "type": "FindAndAttack",
        "target_filter": {
          "type": "LineOfSight"
        }
      }
    ]
  }
}
```
The asset loader would identify `"type": "LineOfSight"`, instantiate a BuilderEntityFilterLineOfSight, call `readConfig`, and then `build` to create the runtime filter object.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderEntityFilterLineOfSight()`. The asset pipeline is responsible for the lifecycle of all builder objects. Manual creation bypasses critical context and validation services.
- **Re-use:** Do not attempt to cache or re-use a builder instance. Each component definition in a configuration file results in a new, single-use builder.

## Data Pipeline
This builder acts as a specific factory node within the larger NPC asset deserialization pipeline.

> Flow:
> NPC Definition (*.json file*) -> Gson Parser -> Asset Loading Service -> **BuilderEntityFilterLineOfSight** -> EntityFilterLineOfSight (Instance) -> NPC Behavior Component

