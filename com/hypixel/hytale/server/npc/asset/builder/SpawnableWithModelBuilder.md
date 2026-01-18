---
description: Architectural reference for SpawnableWithModelBuilder
---

# SpawnableWithModelBuilder

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient (Abstract Base Class)

## Definition
```java
// Signature
public abstract class SpawnableWithModelBuilder<T> extends BuilderBase<T> implements ISpawnableWithModel {
```

## Architecture & Concepts
The SpawnableWithModelBuilder is a foundational abstract class within the server's asset compilation pipeline. It serves as a common template for any builder responsible for constructing game assets that are both spawnable in the world and possess a visual model, such as NPCs, creatures, or complex placeable objects.

Its primary architectural function is to standardize the mechanism for tracking *dynamic dependencies*. These are dependencies that are not known at compile time but are discovered during the parsing of asset definition files. For example, an NPC might be configured to dynamically equip a specific weapon or piece of armor, which are themselves separate assets. This class provides the machinery to register these relationships.

By implementing the ISpawnableWithModel interface, it guarantees a contract that the asset compilation system can rely on to query for dependencies, ensuring that all required models, textures, and other assets are loaded and available before the primary asset is finalized. The generic parameter T represents the concrete asset type that the final builder implementation will produce.

### Lifecycle & Ownership
-   **Creation:** Concrete subclasses are instantiated by the server's asset management system during the initial parsing of asset definition files (e.g., JSON or HOCON configurations). These builders are not intended for direct instantiation by game logic.
-   **Scope:** The lifetime of a SpawnableWithModelBuilder instance is ephemeral and strictly confined to the asset compilation phase. It exists only to parse a definition, report its dependencies, and construct the final asset. It does not persist into the active game session.
-   **Destruction:** Once the corresponding game asset is fully built and registered with the appropriate runtime manager, the builder instance is no longer referenced by the asset system and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class manages a single piece of mutable state: a set of integers named dynamicDependencies. This set is lazily initialized; it is only allocated when the first dynamic dependency is added. This design optimizes for the common case where a spawnable asset has no such dependencies, avoiding unnecessary object allocation.
-   **Thread Safety:** This class is **not thread-safe**. The internal state, an IntOpenHashSet, is not a concurrent collection. The asset building process is designed to be a single-threaded or externally synchronized operation. Concurrent modification of a builder instance from multiple threads will result in race conditions, a corrupted dependency graph, and unpredictable server behavior.

## API Surface
The public API is defined by the ISpawnableWithModel interface, focusing exclusively on dependency management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasDynamicDependencies() | boolean | O(1) | Returns true if any dynamic dependencies have been added. Does not validate the dependencies themselves. |
| addDynamicDependency(int) | void | Amortized O(1) | Registers a dependency, identified by its builder index. Lazily initializes the internal collection on first call. |
| getDynamicDependencies() | IntSet | O(1) | Returns a direct reference to the internal set of dependencies. Returns null if no dependencies exist. |
| clearDynamicDependencies() | void | O(N) | Removes all registered dynamic dependencies from the builder. N is the number of dependencies. |
| isSpawnable() | boolean | O(1) | A contract method that always returns true, signifying that assets produced by this builder can be spawned in the world. |

## Integration Patterns

### Standard Usage
This is an abstract class and cannot be used directly. A developer must extend it to create a builder for a specific asset type. The primary interaction is calling `addDynamicDependency` from within the parsing logic of the concrete implementation.

```java
// Example of a concrete builder for a custom creature
public class DirewolfBuilder extends SpawnableWithModelBuilder<DirewolfAsset> {

    @Override
    public void parse(ConfigurationNode node) {
        // ... parsing logic for Direwolf properties ...

        // If the Direwolf has a configured collar model, add it as a dependency
        if (node.has("collarModel")) {
            String collarAssetName = node.getString("collarModel");
            int dependencyIndex = context.getAssetIndex(collarAssetName);
            addDynamicDependency(dependencyIndex);
        }
    }

    // ... build() method implementation ...
}
```

### Anti-Patterns (Do NOT do this)
-   **External Modification:** Do not retrieve the dependency set via `getDynamicDependencies` and modify it externally. The collection is owned and managed by the builder. External modifications can break the asset system's dependency resolution logic.
-   **Ignoring `BuilderBase`:** This class extends BuilderBase. Subclasses must still correctly implement the `build` and `parse` methods from the parent class to function correctly within the asset pipeline.
-   **Multi-threaded Access:** Never share a single builder instance across multiple threads during the asset parsing phase. All parsing and dependency registration for a single asset must be self-contained or explicitly synchronized.

## Data Pipeline
This builder acts as a critical step in the transformation of a configuration file into a usable in-game asset. It enriches the asset data with a resolved dependency list.

> Flow:
> Asset Definition File (e.g., `direwolf.json`) -> Asset Parser -> `DirewolfBuilder.parse()` -> **SpawnableWithModelBuilder.addDynamicDependency()** -> Asset System Dependency Graph -> `DirewolfBuilder.build()` -> Registered `DirewolfAsset`

