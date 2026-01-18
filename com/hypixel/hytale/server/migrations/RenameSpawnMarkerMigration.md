---
description: Architectural reference for RenameSpawnMarkerMigration
---

# RenameSpawnMarkerMigration

**Package:** com.hypixel.hytale.server.migrations
**Type:** Transient

## Definition
```java
// Signature
public class RenameSpawnMarkerMigration extends EntityMigration<SpawnMarkerEntity> {
```

## Architecture & Concepts
The RenameSpawnMarkerMigration is a specialized component within the server's data migration framework. Its sole responsibility is to handle data consistency when the unique string identifiers for SpawnMarker assets are changed between game versions. This is a common requirement during development to maintain backward compatibility with existing world data.

This class operates as a concrete step in a larger world upgrade process. It targets a specific entity type, SpawnMarkerEntity, and applies a transformation based on a predefined mapping file. The core logic relies on a lookup table that maps legacy string IDs to their modern SpawnMarker asset counterparts.

By extending EntityMigration, it integrates seamlessly into an automated system that iterates over all relevant entities in a world save, applying this migration to each one individually. The parent class provides the necessary boilerplate for entity filtering and version handling, allowing this implementation to focus exclusively on the renaming logic.

### Lifecycle & Ownership
-   **Creation:** An instance is created by a higher-level migration coordinator or world loading service. The constructor requires a file Path pointing to a simple text file that contains the old-ID-to-new-ID mappings.
-   **Scope:** The object's lifetime is ephemeral. It is instantiated for a single migration pass, used to process all relevant entities, and then becomes eligible for garbage collection. It does not persist beyond the scope of the world upgrade task.
-   **Destruction:** The object is destroyed by the Java garbage collector once the migration service that created it releases its reference. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is stateful. Its primary internal state is the idMigrations map, which caches the mapping from old string IDs to new SpawnMarker asset objects. This map is populated once during construction and is treated as immutable thereafter.
-   **Thread Safety:** This class is **not thread-safe**. The internal HashMap is not a concurrent collection. It is designed to be used in a single-threaded migration pipeline. Invoking the migrate method from multiple threads concurrently on the same instance will lead to undefined behavior. The migration framework is responsible for ensuring serialized access.

## API Surface
The public contract is defined by its constructor and the protected `migrate` method, which is the entry point for the migration framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RenameSpawnMarkerMigration(Path) | constructor | O(N) | Creates and initializes the migration. N is the number of lines in the mapping file. Throws IOException via SneakyThrow if the file cannot be read. |
| migrate(SpawnMarkerEntity) | boolean | O(1) | Executes the migration logic for a single entity. Returns true if the entity was modified, false otherwise. This lookup is a constant-time map access. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is instantiated and managed by the server's migration engine during world loading. The engine would identify the need for this migration and execute it as part of a sequence.

```java
// Conceptual usage by a migration service
Path migrationMapFile = serverRoot.resolve("migrations/spawnmarker_rename_v1_to_v2.txt");
EntityMigration<SpawnMarkerEntity> migration = new RenameSpawnMarkerMigration(migrationMapFile);

// The framework would then iterate over all SpawnMarkerEntity instances
for (SpawnMarkerEntity entity : world.getAllEntitiesOfType(SpawnMarkerEntity.class)) {
    boolean wasModified = migration.migrate(entity);
    if (wasModified) {
        // Mark entity as dirty for saving
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not hold a reference to this object after the migration is complete. It is cheap to create and its internal state is specific to the mapping file it was created with.
-   **Manual Invocation:** Do not call the migrate method directly from game systems. Its purpose is strictly for data transformation during an upgrade, not for runtime logic.
-   **Concurrent Access:** Never share an instance of this class across multiple threads. The internal state is not protected by locks.

## Data Pipeline
The flow of data through this component is linear and transformational, designed to update world data in-place during a loading sequence.

> Flow:
> Disk I/O (migration mapping file) -> **RenameSpawnMarkerMigration** (constructor populates internal map) -> World Data Loader (provides SpawnMarkerEntity) -> **RenameSpawnMarkerMigration.migrate()** (transforms entity) -> World Data Loader (persists modified entity)

