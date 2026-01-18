---
description: Architectural reference for BuilderEntityFilterInsideBlock
---

# BuilderEntityFilterInsideBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterInsideBlock extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterInsideBlock is a configuration-time component that implements the **Builder Pattern**. It serves as a factory for creating runtime instances of EntityFilterInsideBlock, which is a specific type of IEntityFilter.

Its primary role is to bridge the gap between static data definitions (typically in JSON files) and live, operational game objects. It is responsible for parsing a JSON configuration block, validating the specified asset references, and holding that configuration until the final runtime object is constructed.

This class decouples the complex and error-prone process of asset parsing and validation from the high-performance runtime logic of the entity filter itself. By resolving the BlockSet asset name into a numerical index during the build phase, it ensures that the runtime filter can perform its checks with maximum efficiency, avoiding costly string lookups during the game loop.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the server's asset loading system when it encounters a corresponding filter type in an NPC's JSON definition file. This process is typically managed by a central registry that maps type names to builder classes.
-   **Scope:** The lifecycle of a BuilderEntityFilterInsideBlock instance is extremely short. It exists only for the duration of the asset parsing and object construction phase for a single NPC.
-   **Destruction:** Once the build method is called and the resulting EntityFilterInsideBlock is passed to the NPC's component system, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** This class is **mutable**. Its primary internal state is the AssetHolder named blockSet, which is populated by the readConfig method. This state is transient and serves only to transfer data from the configuration-parsing stage to the object-building stage.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the server's asset loading sequence. Concurrent calls to readConfig would lead to a race condition and corrupt the internal blockSet state.

## API Surface
The public API is designed for use by the server's asset building framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the runtime EntityFilterInsideBlock instance. |
| readConfig(JsonElement) | Builder | O(N) | Parses the JSON input, validates the required BlockSet asset, and populates internal state. |
| getBlockSet(BuilderSupport) | int | O(log K) | Resolves the configured BlockSet asset name into its integer-based runtime ID. Throws IllegalArgumentException if the asset key is not found. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for use in development tools. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the NPC asset loading pipeline. The framework orchestrates the creation, configuration, and build process.

```java
// Conceptual example of framework usage
BuilderEntityFilterInsideBlock builder = new BuilderEntityFilterInsideBlock();

// The framework provides the relevant JSON snippet
builder.readConfig(npcFilterJson);

// The framework requests the final runtime object
IEntityFilter filter = builder.build(builderSupport);

// The filter is then added to the NPC's component list
npc.addFilter(filter);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually create instances of this class using new. The asset system manages its lifecycle.
-   **State Manipulation:** Do not attempt to modify the internal blockSet field after readConfig has been called. The object's state is considered sealed after parsing.
-   **Instance Re-use:** A builder instance is a one-shot object. Do not attempt to call readConfig multiple times on the same instance to configure different filters.

## Data Pipeline
This builder acts as a transformation step in the NPC asset loading pipeline, converting declarative configuration into an executable object.

> Flow:
> NPC JSON File -> Asset Loading Service -> **BuilderEntityFilterInsideBlock.readConfig()** -> **BuilderEntityFilterInsideBlock.build()** -> EntityFilterInsideBlock (Runtime Object) -> NPC Behavior System

