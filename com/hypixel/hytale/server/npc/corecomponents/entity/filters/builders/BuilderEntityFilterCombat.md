---
description: Architectural reference for BuilderEntityFilterCombat
---

# BuilderEntityFilterCombat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterCombat extends BuilderEntityFilterBase {
```

## Architecture & Concepts

The BuilderEntityFilterCombat class is a factory component within the server-side NPC asset loading pipeline. Its sole responsibility is to translate a declarative JSON configuration into a concrete, executable **EntityFilterCombat** instance. It acts as a bridge between raw data defined by designers and the server's internal game logic.

This builder is a key part of a data-driven design. Instead of hard-coding NPC logic, behaviors are defined in external JSON files. The engine's asset system identifies the filter type from the JSON, instantiates the corresponding builder (in this case, BuilderEntityFilterCombat), and delegates the parsing and object creation process to it.

A critical architectural pattern employed here is **deferred resolution** through the use of Holder objects (e.g., AssetHolder, NumberArrayHolder). The builder does not store the final, resolved values from the JSON. Instead, it stores unresolved references or placeholders. The final values are retrieved on-demand by the created EntityFilterCombat object via the BuilderSupport context. This design decouples the configuration parsing from runtime data resolution, allowing for more flexible and context-aware NPC behaviors.

## Lifecycle & Ownership

-   **Creation:** Instantiated exclusively by the server's asset loading framework. When an NPC behavior asset is parsed, the framework identifies a filter of this type and uses a factory or reflection to create a new BuilderEntityFilterCombat instance. Manual instantiation by developers is strictly forbidden.
-   **Scope:** The builder's lifecycle is intrinsically linked to the EntityFilterCombat instance it produces. The `build` method creates a new filter and passes a reference of the builder itself into the filter's constructor. Consequently, the builder is kept in memory as long as the filter it created exists.
-   **Destruction:** The builder becomes eligible for garbage collection only when the EntityFilterCombat object that holds a reference to it is destroyed. This typically occurs when an NPC's behavior asset is unloaded or the parent NPC entity is removed from the world.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable during the configuration phase, which is initiated and completed within the `readConfig` method call. During this phase, its Holder fields are populated with data from the source JSON. After the `build` method is called, the builder's state should be considered logically immutable, serving as a read-only data source for the filter instance it created.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used within a single-threaded asset loading process. Any attempt to call `readConfig` or `build` from multiple threads on the same instance will result in a corrupted state and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | EntityFilterCombat | O(1) | Constructs and returns the final EntityFilterCombat instance. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the JSON data, populates internal state, and performs validation. N is the number of keys in the JSON object. |
| getSequence(BuilderSupport) | String | O(1) | Resolves and returns the configured attack sequence asset name. Intended for internal use by the created filter. |
| getCombatMode(BuilderSupport) | EntityFilterCombat.Mode | O(1) | Resolves and returns the configured combat mode. Intended for internal use by the created filter. |
| getTimeElapsedRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured time elapsed range. Intended for internal use by the created filter. |

## Integration Patterns

### Standard Usage

This builder is not used directly in game logic code. It is invoked transparently by the asset loading system. A developer or designer interacts with it by defining the corresponding JSON structure in an NPC behavior file.

```json
// Example NPC Behavior JSON Snippet
{
  "type": "Hytale.EntityFilter.Combat",
  "Mode": "Sequence",
  "Sequence": "hytale:boar_charge_attack",
  "TimeElapsedRange": [0.5, 2.0]
}
```

The system processes this configuration as follows:
1.  The asset loader identifies the type `Hytale.EntityFilter.Combat`.
2.  It instantiates a `BuilderEntityFilterCombat`.
3.  It calls `builder.readConfig(jsonData)`.
4.  It calls `builder.build(builderSupport)` to get the final `IEntityFilter`.
5.  The resulting filter is integrated into the NPC's behavior tree or logic controller.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BuilderEntityFilterCombat()`. The asset system is responsible for managing the lifecycle of builders. Direct creation bypasses the asset pipeline and will result in non-functional components.
-   **State Re-configuration:** Do not call `readConfig` on a builder instance that has already been used to build an object. Each filter requires a unique, dedicated builder instance.
-   **Builder Reuse:** Do not attempt to cache and reuse a single builder instance to create multiple, distinct filters. This will lead to state corruption and unpredictable behavior, as the second filter will be built with a mix of old and new configuration data.

## Data Pipeline

The flow of data from configuration to a live game object is linear and unidirectional.

> Flow:
> NPC Behavior JSON File -> Asset Parsing Service -> **BuilderEntityFilterCombat.readConfig()** -> **BuilderEntityFilterCombat.build()** -> Live EntityFilterCombat Instance -> NPC Behavior Tree Evaluation

