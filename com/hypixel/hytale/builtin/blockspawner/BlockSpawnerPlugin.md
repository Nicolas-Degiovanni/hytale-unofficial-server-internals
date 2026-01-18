---
description: Architectural reference for BlockSpawnerPlugin
---

# BlockSpawnerPlugin

**Package:** com.hypixel.hytale.builtin.blockspawner
**Type:** Singleton (Plugin)

## Definition
```java
// Signature
public class BlockSpawnerPlugin extends JavaPlugin {
```

## Architecture & Concepts

The BlockSpawnerPlugin is a self-contained server module that implements the "Block Spawner" game mechanic. It operates on a principle of **procedural transformation**: a temporary spawner block, when loaded into the world, is deterministically replaced by a final, permanent block based on a set of predefined rules.

This plugin is a prime example of Hytale's Entity Component System (ECS) architecture in practice. It does not contain a traditional update loop. Instead, it registers its own data structures and logic systems with the core engine, which then invokes the plugin's code at the correct time.

The key architectural components are:
*   **BlockSpawner (Component):** A simple data component attached to a block entity. It holds an ID that references a BlockSpawnerTable asset. Its presence on an entity is the trigger for the entire process.
*   **BlockSpawnerTable (Asset):** A data asset that defines the possible outcomes for a spawner. It contains a weighted list of potential blocks to spawn, along with rules for their rotation.
*   **BlockSpawnerSystem (System):** The logical core of the plugin. This ECS system queries for all entities that have a BlockSpawner component. When one is added to the world, the system executes the transformation logic.
*   **MigrateBlockSpawner (System):** A deprecated data migration system. Its sole purpose is to ensure backward compatibility by upgrading older, unstructured spawner data in world saves to the modern component-based format.

The entire process is deterministic, relying on the world seed and the block's absolute coordinates. This ensures that a world will always generate identically, a critical feature for shareable, procedurally generated content.

### Lifecycle & Ownership
-   **Creation:** The BlockSpawnerPlugin is instantiated exactly once by the server's plugin loader during the server bootstrap sequence. The static INSTANCE field is set within the constructor, enforcing the singleton pattern.
-   **Scope:** Application-scoped. The plugin instance persists for the entire lifetime of the server process. Its registered systems and components become an integral part of the server's core game logic.
-   **Destruction:** The instance is dereferenced and garbage collected during server shutdown when the plugin registry is cleared. No explicit destruction logic is required.

## Internal State & Concurrency
-   **State:** The plugin object itself is effectively stateless after initialization. It holds a reference to its registered ComponentType, which is assigned during the `setup` phase and never changes. The actual state relevant to this system (e.g., which blocks are spawners) is external, stored in `BlockSpawner` components attached to entities within the world's `ChunkStore`.
-   **Thread Safety:** The plugin's initialization via the `setup` method occurs in a single-threaded context during server startup, making it inherently safe. The primary logic within `BlockSpawnerSystem` is invoked by the ECS framework during the world update tick.

    **WARNING:** The system's use of a `CommandBuffer` is a critical concurrency control mechanism. World state modifications (removing the spawner entity, setting the new block) are not performed immediately. Instead, they are queued as commands. This prevents `ConcurrentModificationException` and other race conditions that would arise from modifying the entity list while it is being iterated. The command buffer is flushed at a safe point later in the tick cycle.

## API Surface

The public API is minimal, as interaction is primarily data-driven through assets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static BlockSpawnerPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| getBlockSpawnerComponentType() | ComponentType | O(1) | Returns the registered component type for BlockSpawner. Used for advanced ECS queries. |

## Integration Patterns

### Standard Usage

Direct interaction with this plugin in code is rare. The primary integration path is through data definition and world editing. A content creator defines a spawner asset and places a corresponding block entity in the world.

However, if another plugin needed to programmatically identify spawner blocks, it would use the registered component type.

```java
// Example: A custom system that logs the location of all spawners
// This code would exist in a different plugin's system.

// 1. Get the component type from the singleton
ComponentType<ChunkStore, BlockSpawner> spawnerType = BlockSpawnerPlugin.get().getBlockSpawnerComponentType();

// 2. Query for entities with that component
Query<ChunkStore> spawnerQuery = Query.is(spawnerType);

// 3. In the system's update/onEntityAdded method...
Store<ChunkStore> store = ...;
store.forEach(spawnerQuery, (ref, holder) -> {
    // This entity is a spawner, perform custom logic
    System.out.println("Found a spawner at entity: " + ref.getId());
});
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BlockSpawnerPlugin()`. The server's plugin loader is solely responsible for its creation. Attempting to do so will break the singleton pattern and lead to an uninitialized, non-functional object.
-   **Direct World Modification:** Do not attempt to replicate the logic of `BlockSpawnerSystem` by calling `world.setBlock` or `store.removeEntity` directly from within an ECS system's iteration loop. This bypasses the `CommandBuffer` and will lead to severe concurrency issues and world corruption.
-   **Reliance on Timing:** Do not write code that assumes a spawner will be transformed within the same tick it is created programmatically. The transformation is handled by a system that runs at a specific point in the game loop; the change will not be visible until a subsequent tick.

## Data Pipeline

The transformation of a spawner block follows a well-defined, reactive data pipeline orchestrated by the Entity Component System.

> Flow:
> World Chunk Load -> Entity Deserialization -> **BlockSpawner** Component Added -> `ChunkStoreRegistry` Notifies System -> **BlockSpawnerSystem.onEntityAdded** -> Deterministic Hash Calculation -> Asset Lookup -> CommandBuffer Population -> End of Tick -> CommandBuffer Flush -> Spawner Entity Removed & New Block Placed in `WorldChunk` -> Client State Update

