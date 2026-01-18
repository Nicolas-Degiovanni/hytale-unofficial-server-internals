---
description: Architectural reference for BlockMigration
---

# BlockMigration

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Managed Asset / Data Model

## Definition
```java
// Signature
public class BlockMigration implements JsonAssetWithMap<Integer, DefaultAssetMap<Integer, BlockMigration>> {
```

## Architecture & Concepts

The BlockMigration class is a data-driven configuration asset that provides a backward-compatibility layer for block types. Its primary function is to define deterministic mappings from old, legacy block identifiers to their modern equivalents. This system is critical for world upgrades, ensuring that worlds created in older versions of the game can be loaded correctly in newer versions where block names or internal keys may have changed.

Each BlockMigration instance represents a specific version or set of migration rules, identified by an integer ID. It contains two distinct mapping strategies:

1.  **Direct Migrations:** A map for explicit, one-to-one remapping of block keys. This is used for simple renames, for example, `hytale:old_wood` -> `hytale:birch_planks`.
2.  **Name Migrations:** A secondary map for more complex or pattern-based renames.

Instances of this class are not meant to be created programmatically. Instead, they are deserialized from JSON configuration files by the engine's core asset loading system. The static `CODEC` field is a declarative schema that instructs the `AssetStore` how to parse the JSON and populate a BlockMigration object. This decouples the migration data from the game logic, allowing developers to update migration rules by simply changing a configuration file.

The class acts as a client to the global `AssetRegistry`, retrieving a shared, cached map of all loaded BlockMigration assets via the static `getAssetMap` method.

## Lifecycle & Ownership

-   **Creation:** Instances are instantiated by the `AssetStore` service during the server's initial asset loading phase. The public static `CODEC` field is consumed by the asset loader to construct objects from corresponding JSON files on disk. Manual instantiation is strongly discouraged.
-   **Scope:** A collection of all BlockMigration assets persists for the entire server session. This collection is loaded once at startup and cached in the static `ASSET_MAP` field upon first access. Individual instances are effectively immortal for the duration of the server's runtime.
-   **Destruction:** The static `ASSET_MAP` and all contained BlockMigration instances are eligible for garbage collection upon server shutdown when the `AssetRegistry` is cleared.

## Internal State & Concurrency

-   **State:** The state of a BlockMigration instance is effectively immutable after it has been deserialized from its source asset file. The `directMigrations` and `nameMigrations` maps are populated once during creation and are not subsequently modified. The class serves as a read-only data container.
-   **Thread Safety:** Instance methods such as `getMigration` are thread-safe, as they only perform read operations on the internal, unchanging maps.

    **WARNING:** The static `getAssetMap` method employs a non-thread-safe lazy initialization pattern. If multiple threads attempt to access the asset map for the first time concurrently, it may result in multiple lookups against the `AssetRegistry`. While the `AssetRegistry` itself is expected to be thread-safe and return the same logical map, this pattern is inefficient and can lead to unpredictable behavior in highly concurrent startup environments. Access should ideally be synchronized externally if this scenario is possible.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static DefaultAssetMap | O(1) | Retrieves the global, cached map of all BlockMigration assets, keyed by their integer ID. The first call incurs a higher cost due to registry lookup. |
| getMigration(String) | String | O(1) | Resolves a new block key from an old one. It prioritizes a direct migration, falls back to a name migration, and returns the original key if no mapping exists. |
| getId() | Integer | O(1) | Returns the unique integer identifier for this specific set of migration rules. |

## Integration Patterns

### Standard Usage

The primary integration pattern involves retrieving the global map of migrations, selecting the appropriate ruleset by its ID (e.g., a version number stored with world data), and then using that ruleset to translate block keys during chunk loading or world conversion processes.

```java
// How a developer should normally use this
// 1. Retrieve the central map of all migration configurations
DefaultAssetMap<Integer, BlockMigration> allMigrations = BlockMigration.getAssetMap();

// 2. Get the specific rules for the world's data version (e.g., version 4)
int worldDataVersion = 4;
BlockMigration migrationRules = allMigrations.get(worldDataVersion);

// 3. Apply the migration rules during data processing
if (migrationRules != null) {
    String oldBlockKey = "hytale:grass_old";
    String newBlockKey = migrationRules.getMigration(oldBlockKey);
    // newBlockKey will be the migrated value, or the original if no rule exists
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlockMigration()`. This creates an unmanaged, empty object that is not part of the asset system. It will contain no migration rules and will not be accessible via `getAssetMap`.
-   **State Mutation:** Do not attempt to modify the maps returned by `getDirectMigrations` or `getNameMigrations`. These are intended to be read-only, and mutating them can lead to undefined behavior across the server.
-   **Concurrent Initialization:** Avoid calling `getAssetMap` from multiple threads simultaneously during server startup without external synchronization. This can trigger the race condition described in the Concurrency section.

## Data Pipeline

The flow of data from configuration file to in-game use is managed entirely by the Hytale asset system.

> Flow:
> `block_migration.json` on Disk -> Server Asset Loader -> **BlockMigration.CODEC** -> Deserialized **BlockMigration** Instance -> Stored in AssetRegistry -> `BlockMigration.getAssetMap()` -> World Upgrade System

