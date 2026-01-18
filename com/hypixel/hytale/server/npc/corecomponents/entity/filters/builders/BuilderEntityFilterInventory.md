---
description: Architectural reference for BuilderEntityFilterInventory
---

# BuilderEntityFilterInventory

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderEntityFilterInventory extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterInventory class is a key component in the server's data-driven NPC behavior system. It functions as a specialized factory and deserializer, translating static configuration data from JSON files into a live, executable game logic component—specifically, an EntityFilterInventory.

This class embodies the **Builder Pattern**. Its primary responsibility is to parse a JSON definition, validate its parameters, and hold that configuration until the engine requests the final, usable filter object. It acts as the bridge between the declarative world of asset configuration and the imperative world of the game simulation. By abstracting the construction logic, the system allows game designers to define complex inventory-based conditions for NPCs without writing any Java code.

The builder operates on three primary axes of inventory state:
1.  **Item Presence:** Matching items by name or glob patterns.
2.  **Item Quantity:** Verifying that the count of matched items falls within a specified range.
3.  **Inventory Capacity:** Checking for a number of free slots within a given range.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderEntityFilterInventory is not created directly by game logic developers. It is instantiated by the server's central asset and configuration management system when it encounters a JSON component of the corresponding type (e.g., `hytale:entity_filter_inventory`).

-   **Scope:** The builder object is transient. It exists to process a specific JSON definition. Its state is populated once during the `readConfig` phase. It may be held in memory by the asset system as part of a cached, pre-parsed configuration, but its role in any single filter's creation is short-lived.

-   **Destruction:** The object is managed by the Java garbage collector. Once the final EntityFilterInventory object has been built and the asset system no longer holds a reference to the builder, it becomes eligible for collection. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The internal state of this builder is **highly mutable**. The `readConfig` method directly modifies the internal AssetArrayHolder and NumberArrayHolder fields. This state represents a parsed, but not yet fully resolved, version of the JSON configuration. The final resolution of asset names and values is deferred until the `build` method is called, which requires a BuilderSupport context.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The builder pattern assumes a single-threaded construction context. All operations, particularly `readConfig`, must be confined to the server's main asset loading thread to prevent data corruption and race conditions. The object it produces, EntityFilterInventory, may have different concurrency guarantees.

## API Surface
The public API is designed for use by the configuration system, not for general-purpose invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterInventory instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the input JSON, populating and validating the internal state. N is the number of elements in the JSON. |
| getItems(BuilderSupport) | String[] | O(1) | Resolves and returns the configured item patterns. Intended for use by the constructed filter object. |
| getCount(BuilderSupport) | int[] | O(1) | Resolves and returns the configured item count range. Intended for use by the constructed filter object. |
| getFreeSlotsRange(BuilderSupport) | int[] | O(1) | Resolves and returns the configured free slot range. Intended for use by the constructed filter object. |

## Integration Patterns

### Standard Usage
A developer or game designer does not typically interact with this class directly in Java. Instead, they define the filter's properties in a JSON asset file. The engine handles the lifecycle of the builder internally.

**Conceptual JSON Asset:**
```json
{
  "type": "hytale:entity_filter_inventory",
  "items": ["hytale:iron_sword", "hytale:torch_*"],
  "countRange": [1, 5],
  "freeSlotRange": [10, 99]
}
```

**Engine's Internal Invocation (Simplified):**
```java
// The engine's asset loader finds the appropriate builder for the JSON type
BuilderEntityFilterInventory builder = assetRegistry.getBuilderFor("hytale:entity_filter_inventory");

// The engine passes the JSON data to the builder
JsonElement configData = parseJsonFile("my_npc_behavior.json");
builder.readConfig(configData);

// Later, when an NPC needs the filter, the engine builds the final object
BuilderSupport supportContext = engine.createSupportContext();
IEntityFilter filter = builder.build(supportContext);

// The filter is now ready to be used by the NPC's AI
boolean result = filter.evaluate(targetEntity);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BuilderEntityFilterInventory()`. The construction of builders is managed by the asset framework to ensure proper registration and lifecycle.

-   **State Re-use:** Do not call `readConfig` multiple times on the same instance to configure different filters. A builder instance is meant to represent a single, specific configuration.

-   **Concurrent Modification:** Never access a builder instance from multiple threads. The internal state is not protected by locks, and concurrent calls to `readConfig` or `build` will lead to unpredictable and erroneous behavior.

## Data Pipeline
The flow of data from configuration to execution is linear and unidirectional.

> Flow:
> JSON Asset File -> Server Asset Loader -> **BuilderEntityFilterInventory.readConfig()** -> In-Memory Config -> **BuilderEntityFilterInventory.build()** -> EntityFilterInventory Instance -> NPC Behavior Tree Evaluation

