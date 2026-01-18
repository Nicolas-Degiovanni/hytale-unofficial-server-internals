---
description: Architectural reference for ChunkSectionMigrationSystem
---

# ChunkSectionMigrationSystem

**Package:** com.hypixel.hytale.server.core.modules.migrations
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class ChunkSectionMigrationSystem extends HolderSystem<ChunkStore> {
```

## Architecture & Concepts
The ChunkSectionMigrationSystem is an abstract base class that forms the core of the server's world data upgrade and migration framework. It establishes a contract for any process that needs to perform a structural transformation on persisted chunk data, typically when a server update introduces breaking changes to the world format.

This class acts as a specialized, single-purpose data transformer. Its design is tightly coupled to the world storage layer, as indicated by its generic parameterization with ChunkStore. Concrete implementations of this class contain the specific logic to upgrade chunk data from one version to another.

These systems are discovered and executed by a higher-level migration coordinator during the world loading process. This ensures that all world data is brought to a consistent, up-to-date state before being used by the live game server, preventing data corruption and compatibility errors.

### Lifecycle & Ownership
- **Creation:** Concrete implementations are discovered and instantiated by a central MigrationModule or WorldLoader service at server startup or during the initial load of a world. They are not created on-demand during active gameplay.
- **Scope:** The lifecycle of a migration system instance is extremely short and transient. It is created to perform a single migration task on a specific set of data and is then immediately discarded.
- **Destruction:** The object is eligible for garbage collection as soon as its primary migration method completes. It holds no persistent state and is not retained by any long-lived service.

## Internal State & Concurrency
- **State:** This abstract class is stateless. Concrete implementations are **strongly expected** to be stateless as well. All necessary data for the operation should be passed into the execution method, and any temporary state should be confined to local method variables. Storing data at the instance level is a severe anti-pattern.
- **Thread Safety:** Implementations of this class are **not thread-safe** and are not designed to be. The migration framework guarantees that each migration system will be executed sequentially within a single, isolated thread dedicated to the world upgrade process. Attempting to access a ChunkStore from multiple threads or migration systems concurrently will lead to world corruption.

## API Surface
As an abstract class, its primary API is the contract it forces upon its children. While the class itself is empty, it inherits from HolderSystem and implies a primary execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (abstract) run() | void | O(N) | *Inherited/Implied.* The core execution method containing the migration logic. N is the size of the chunk data being processed. |
| getHolder() | ChunkStore | O(1) | *Inherited.* Retrieves the ChunkStore instance that this migration system is designated to operate upon. |

## Integration Patterns

### Standard Usage
A developer never interacts with this class directly. Instead, they create a concrete implementation to define a new data migration path. The system is then automatically discovered and executed by the server's migration coordinator.

```java
// Define a specific migration to upgrade from data version 4 to 5.
// The framework will automatically discover and run this class when loading a chunk with version 4.
public class V4ToV5ChunkStructureUpgrade extends ChunkSectionMigrationSystem {

    @Override
    public void run() {
        ChunkStore store = getHolder();
        if (store.getDataVersion() != 4) {
            // It is critical to validate the version before proceeding.
            return;
        }

        // 1. Read old data structures from the store.
        OldDataStructure oldData = store.read("legacy_block_data");

        // 2. Perform transformation logic.
        NewDataStructure newData = transformToNewStructure(oldData);

        // 3. Write new data structures and update the version.
        store.write("block_data", newData);
        store.remove("legacy_block_data");
        store.setDataVersion(5);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Execution:** Never instantiate and run a migration system manually. The migration coordinator is responsible for executing systems in the correct order and ensuring data integrity. Bypassing it can corrupt world data.
- **Stateful Implementations:** Do not add fields to your implementation to store state between method calls. Each migration is a self-contained, idempotent operation.
- **Ignoring Version Checks:** A migration must *always* check the current data version of the ChunkStore before acting. Applying a migration to data that is already migrated or is from a future version will cause irreversible damage.

## Data Pipeline
This component functions as a critical step in the world data loading pipeline. It is invoked only when outdated data is detected, acting as a conditional transformation stage.

> Flow:
> World Loader -> Detects Outdated Chunk Version -> Migration Coordinator -> **ChunkSectionMigrationSystem (Implementation)** -> Reads & Transforms Data -> Writes to ChunkStore -> World Loader Resumes

