---
description: Architectural reference for LegacyModule
---

# LegacyModule

**Package:** com.hypixel.hytale.server.core.modules
**Type:** Singleton

## Definition
```java
// Signature
public class LegacyModule extends JavaPlugin {
```

## Architecture & Concepts
The LegacyModule is a foundational server plugin responsible for bootstrapping the world's Entity Component System (ECS) representation. It acts as the central registry for all core data structures that define a game world chunk, such as BlockChunk, EntityChunk, and ChunkColumn.

Its primary function is to define and register these fundamental **ComponentTypes** with the server's ChunkStoreRegistry during the server startup sequence. By doing so, it establishes the schema for how world data is stored, loaded, and manipulated in memory.

Furthermore, this module is critical for maintaining backward compatibility of world saves. It registers specialized migration systems, such as MigrateLegacyBlockStateChunkSystem, which automatically detect and upgrade outdated chunk data formats to the current standard upon loading. This responsibility makes it a cornerstone of the world persistence layer, ensuring smooth transitions between game versions without data loss.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's plugin loading mechanism during the initial bootstrap phase. The constructor is invoked with a JavaPluginInit context, which provides access to essential server registries. The singleton instance is set within the constructor.

- **Scope:** The LegacyModule is a session-scoped singleton. It persists for the entire lifetime of the server process.

- **Destruction:** The object is dereferenced and becomes eligible for garbage collection during server shutdown when the plugin container is dismantled. No explicit cleanup logic is present or required.

## Internal State & Concurrency
- **State:** The internal state consists of a collection of ComponentType fields (e.g., worldChunkComponentType, blockChunkComponentType). This state is mutable exclusively during the execution of the setup method. After this initialization phase, the state becomes effectively immutable and is only read via public getters.

- **Thread Safety:** This class is thread-safe under normal operating conditions. The state-mutating setup method is guaranteed by the server to be called only once from a single main thread during startup. All subsequent access to its state via the public getters is read-only, preventing race conditions.

## API Surface
The public contract is designed for service discovery, providing access to the canonical ComponentType definitions for world data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | LegacyModule | O(1) | Statically retrieves the singleton instance of the module. |
| getWorldChunkComponentType() | ComponentType | O(1) | Returns the registered type definition for WorldChunk. |
| getBlockChunkComponentType() | ComponentType | O(1) | Returns the registered type definition for BlockChunk. |
| getEntityChunkComponentType() | ComponentType | O(1) | Returns the registered type definition for EntityChunk. |
| getBlockComponentChunkComponentType() | ComponentType | O(1) | Returns the registered type definition for BlockComponentChunk. |
| getChunkColumnComponentType() | ComponentType | O(1) | Returns the registered type definition for ChunkColumn. |
| getBlockSectionComponentType() | ComponentType | O(1) | Returns the registered type definition for BlockSection. |

## Integration Patterns

### Standard Usage
Systems that need to interact with the chunk ECS must first retrieve the correct ComponentType from this module. This ensures that all parts of the server are operating on the same, centrally-defined data types.

```java
// Retrieve the canonical ComponentType for Block Sections
LegacyModule module = LegacyModule.get();
ComponentType<ChunkStore, BlockSection> blockSectionType = module.getBlockSectionComponentType();

// Use the type to query or modify a chunk entity
Holder<ChunkStore> chunkEntity = ...;
if (chunkEntity.hasComponent(blockSectionType)) {
    BlockSection section = chunkEntity.getComponent(blockSectionType);
    // ... operate on the section
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new LegacyModule(). The server's plugin loader is solely responsible for its creation. Manual instantiation will break the singleton pattern and lead to a detached, non-functional instance, likely causing NullPointerExceptions or registry corruption.

- **Premature Access:** Do not call the static get method or any getters before the server's plugin initialization phase is complete. Accessing the module before its setup method has run will result in null ComponentType references and will cause critical system failures.

## Data Pipeline
LegacyModule does not directly process a continuous stream of data. Instead, it configures the systems that form the data pipeline for loading and migrating chunk data. The most critical pipeline it establishes is for backward compatibility.

> **Legacy Chunk Migration Flow:**
>
> Raw Chunk Data from Disk -> Server Deserializer -> ChunkStore creates an entity with a legacy component (e.g., LegacyBlockStateChunk) -> **MigrateLegacyBlockStateChunkSystem** (registered by LegacyModule) is triggered by the ECS -> The system removes the legacy component and adds the modern equivalent (BlockComponentChunk) -> The chunk entity is now in the current, usable format.

