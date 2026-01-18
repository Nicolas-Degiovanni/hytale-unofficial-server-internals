---
description: Architectural reference for BlockModule
---

# BlockModule

**Package:** com.hypixel.hytale.server.core.modules.block
**Type:** Singleton

## Definition
```java
// Signature
public class BlockModule extends JavaPlugin {
```

## Architecture & Concepts
The BlockModule is a foundational server plugin that governs the lifecycle and data management of all stateful blocks. It acts as the primary bridge between the static, grid-based world representation (chunks of block IDs) and the dynamic Entity Component System (ECS).

Its core responsibility is to define and manage the schema for "Block Entities"—ECS entities that are spatially bound to a specific block coordinate and hold additional data or logic. This allows simple blocks like stone to remain as raw data in a chunk, while complex blocks like chests, signs, or custom machinery can participate in the full ECS simulation.

The module operates by:
1.  **Registering Components:** During server initialization, it registers critical ECS components such as LaunchPad, RespawnBlock, and the internal BlockStateInfo with the server's ChunkStore registry. This establishes the data "shape" for all special blocks.
2.  **Registering Systems:** It registers ECS systems, most notably the BlockStateInfoRefSystem, which automatically manages the bidirectional relationship between a chunk and its associated block entities.
3.  **Automating Instantiation:** It listens for the ChunkPreLoadProcessEvent to scan newly generated chunks. For any block type configured to have a block entity or special state, this module automatically creates the corresponding ECS entity and attaches the necessary components. This is the primary mechanism for populating the world with interactive blocks.
4.  **Providing a Query API:** It exposes a public API for other game systems to query for block entities and their components at a given world coordinate.

This module is central to world interactivity and persistence. Any system that needs to interact with blocks beyond their basic type must integrate with the BlockModule.

## Lifecycle & Ownership
- **Creation:** The BlockModule is instantiated once by the server's plugin loader during the bootstrap sequence. The static singleton instance is set within the constructor, making it globally accessible via the static `get` method.
- **Scope:** Application-scoped. The BlockModule persists for the entire lifetime of the server process. Its registered components and systems become a permanent part of the server's ECS schema.
- **Destruction:** The module is only destroyed when the server shuts down. There are no explicit cleanup or teardown procedures; it relies on the termination of the Java Virtual Machine.

## Internal State & Concurrency
- **State:** The BlockModule instance itself is effectively stateless after initialization. It holds immutable references to ECS type handles (e.g., ComponentType, ResourceType) which are assigned once in the `setup` method. The actual state it manages—the block entities and their components—is stored externally within the world's `ChunkStore` and is not held by the module instance.
- **Thread Safety:** The module is thread-safe. Initialization occurs synchronously on the main server thread. Public API methods like `getComponent` and `getBlockEntity` are safe to call from any thread, as they delegate concurrency control to the underlying `ChunkStore` and `World` objects. The `onChunkPreLoadProcessEnsureBlockEntity` event handler is designed to be executed by world generation worker threads and operates on chunk data within a safe, well-defined context.

## API Surface
The public API provides methods to query for block entities and their components. Getters for component types are primarily for use in other ECS systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | BlockModule | O(1) | Retrieves the global singleton instance of the module. |
| getBlockEntity(world, x, y, z) | Ref<ChunkStore> | O(log N) | Fetches a reference to the ECS entity at the specified block coordinates. Returns null if no entity exists. Complexity is dominated by chunk lookup. |
| getComponent(type, world, x, y, z) | T | O(log N) | A convenience method to directly retrieve a specific component from a block entity at the given coordinates. Returns null if the entity or component does not exist. |
| ensureBlockEntity(chunk, x, y, z) | Ref<ChunkStore> | O(log N) | **Deprecated.** Creates a block entity on-demand. Use of this method is heavily discouraged as it bypasses the standard world-generation pipeline. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is to query for existing block entity data. Direct creation is handled automatically by the engine during world generation.

```java
// How a developer should normally use this
BlockModule blockModule = BlockModule.get();
World world = ...; // Obtain the world context

// Retrieve a specific component from a block at (100, 64, 200)
LaunchPad launchPad = blockModule.getComponent(LaunchPad.getComponentType(), world, 100, 64, 200);

if (launchPad != null) {
    // Interact with the LaunchPad component
    launchPad.activate();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockModule()`. The server's plugin loader is solely responsible for its creation. Always use the static `BlockModule.get()` accessor.
- **Manual Entity Creation:** Do not use the deprecated `ensureBlockEntity` method. Relying on this can cause race conditions with the world generator and lead to duplicate entities or inconsistent world state. Block entity creation should be configured declaratively on the BlockType asset.
- **State Storage:** Do not add mutable state fields to the BlockModule class itself. It is a singleton manager, not a data container. All state must be stored in ECS components.

## Data Pipeline
The BlockModule's most critical function is the data pipeline that transforms raw block data into interactive ECS entities during world generation.

> Flow:
> World Generator produces raw chunk data -> Server fires `ChunkPreLoadProcessEvent` -> **BlockModule** listener `onChunkPreLoadProcessEnsureBlockEntity` is invoked -> The listener iterates all block coordinates in the new chunk -> For each block, it checks the `BlockType` asset definition -> If the `BlockType` requires a block entity, a new ECS entity is created -> A `BlockStateInfo` component is added to the entity, linking it to its coordinate -> The `BlockStateInfoRefSystem` detects the new entity via its query -> The system updates the `BlockComponentChunk` with a reference to the new entity, completing the link.

