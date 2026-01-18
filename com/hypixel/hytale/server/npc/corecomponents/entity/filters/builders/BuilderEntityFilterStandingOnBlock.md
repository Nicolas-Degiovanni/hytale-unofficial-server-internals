---
description: Architectural reference for BuilderEntityFilterStandingOnBlock
---

# BuilderEntityFilterStandingOnBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderEntityFilterStandingOnBlock extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterStandingOnBlock is a key component in the server's data-driven NPC (Non-Player Character) configuration system. It functions as a specialized factory responsible for deserializing a JSON configuration snippet and constructing an instance of EntityFilterStandingOnBlock.

This class embodies the **Builder Pattern**, acting as an intermediary between raw configuration data (JSON) and a concrete, executable game logic object (an IEntityFilter). Its primary role is to parse, validate, and prepare the necessary data—specifically, a reference to a BlockSet asset—before the final filter object is instantiated.

This architecture decouples game logic from its configuration, enabling designers to define complex entity targeting behaviors in data files without modifying or recompiling Java code. This builder is discovered and utilized by a higher-level configuration loader, which dynamically instantiates it based on the filter type specified in an NPC definition file.

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by a higher-level configuration parsing system (e.g., an EntityFilterFactory) when it encounters the corresponding filter type name within an NPC's JSON definition. It is not managed by a dependency injection container or service registry.
-   **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single filter definition. After its build method is called and the resulting EntityFilterStandingOnBlock is returned, the builder object is no longer referenced and becomes eligible for garbage collection.
-   **Destruction:** Handled by the standard Java garbage collector. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
-   **State:** This object is **highly mutable and stateful**. Its primary internal state is the `blockSet` AssetHolder, which is populated during the call to `readConfig`. This state is transient and serves only to transfer configuration from the JSON source to the constructor of the final filter object.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured via `readConfig`, and used via `build` in a single, synchronous sequence within the asset loading pipeline. Concurrent access would lead to race conditions on its internal `blockSet` field.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterStandingOnBlock instance. |
| readConfig(JsonElement) | Builder | O(N) | Deserializes the JSON data, populating the internal state. Validates the required BlockSet asset. |
| getBlockSet(BuilderSupport) | int | O(log K) | Resolves the configured BlockSet asset name into its runtime integer ID. Throws IllegalArgumentException if the asset key is not found. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description of the filter's purpose. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development stability status of this builder, indicating if it is safe for production use. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by most developers. It is invoked by the server's configuration loading machinery. The conceptual flow within that system is as follows.

```java
// Pseudo-code for the engine's config loader
JsonElement filterConfig = parseNpcDefinitionFile(".../npc.json");
String filterType = filterConfig.get("type").getAsString(); // e.g., "StandingOnBlock"

// The engine dynamically finds and creates the correct builder
BuilderEntityFilterBase builder = FilterBuilderRegistry.createBuilder(filterType);

// The builder is configured from JSON and then used to create the final object
builder.readConfig(filterConfig);
IEntityFilter finalFilter = builder.build(builderSupport);

// The builder is now discarded.
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not cache and reuse a builder instance. It is stateful and designed for a single build operation. Reusing it will result in subsequent filters being created with stale data from the first configuration.
-   **Manual Instantiation:** Avoid `new BuilderEntityFilterStandingOnBlock()`. The configuration system is responsible for locating and instantiating the correct builder class.
-   **Out-of-Order Calls:** Do not call `build` or `getBlockSet` before `readConfig` has been successfully executed. Doing so will result in exceptions or incorrect behavior as the internal state will not be properly initialized.

## Data Pipeline
The builder acts as a transformation stage in the NPC asset loading pipeline, converting a declarative JSON structure into an executable Java object.

> Flow:
> NPC Definition File (JSON) -> Server Config Loader -> **BuilderEntityFilterStandingOnBlock.readConfig()** -> **BuilderEntityFilterStandingOnBlock.build()** -> EntityFilterStandingOnBlock Instance -> NPC Behavior Tree

