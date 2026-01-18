---
description: Architectural reference for ChunkSystems
---

# ChunkSystems

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.systems
**Type:** Utility / System Container

## Definition
```java
// Signature
public class ChunkSystems {
    // Contains only static nested System classes
}
```

## Architecture & Concepts
The ChunkSystems class is not a traditional object but a static container for a suite of related Entity Component Systems (ECS). These systems collectively manage the server-side lifecycle of world chunks, from initial creation and loading to the replication of block-level changes to clients.

This class embodies a core principle of the Hytale server architecture: decomposing complex world management tasks into small, discrete, and stateless systems that operate on entities based on their component composition. The systems within ChunkSystems form a reactive pipeline, where the addition or removal of one component on a chunk entity triggers a cascade of subsequent systems to further define and initialize it.

The primary entities managed by this suite are:
-   **WorldChunk Entity**: The root entity representing a 16x256x16 column of blocks in the world.
-   **ChunkSection Entity**: A child entity representing a 16x16x16 sub-chunk of a WorldChunk.

The overall responsibility of this module is to translate the abstract concept of a "chunk" into a fully realized set of component-based entities that the rest of the server can interact with, and to ensure that modifications to this state are efficiently propagated to all subscribed clients.

### Lifecycle & Ownership
-   **Creation:** The ChunkSystems class itself is never instantiated. It is a static-only container. The nested system classes within it are instantiated by the ECS framework during server initialization and registered with the primary `Store`.
-   **Scope:** The systems persist for the entire server session. They are stateless services that are invoked by the ECS engine each tick or upon entity changes.
-   **Destruction:** The systems are destroyed when the server shuts down and the ECS `Store` is cleared.

## Internal State & Concurrency
The ChunkSystems container class is stateless. The contained systems are designed to be highly concurrent and operate on isolated data.

-   **State:** The systems themselves are stateless. All state is held within the components they operate on, such as BlockSection or ChunkColumn. They read from components, process data, and write results back to components via a CommandBuffer, ensuring transactional integrity within a single tick.
-   **Thread Safety:** The systems are designed for thread safety within the ECS framework's execution model. The ReplicateChanges system is explicitly designed for parallel execution across multiple archetype chunks, as indicated by its `isParallel` method. This is critical for performance, as block change replication is a high-frequency, high-cost operation on a populated server.

## API Surface
The public API of ChunkSystems is the collection of systems it provides for registration with the ECS engine. These are not meant to be called directly by application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OnNewChunk | HolderSystem | O(1) | Bootstraps a new chunk entity by adding a ChunkColumn component. |
| OnChunkLoad | RefSystem | O(N) | Populates a ChunkColumn with its child ChunkSection entities. N is the number of sections. |
| EnsureBlockSection | HolderSystem | O(1) | Guarantees that a ChunkSection entity has a BlockSection for storing block data. |
| LoadBlockSection | HolderSystem | O(1) | Marks a BlockSection as loaded. |
| OnNonTicking | RefChangeSystem | O(N) | Propagates the NonTicking state from a chunk column to its sections. |
| ReplicateChanges | EntityTickingSystem | O(M) | Detects and replicates block changes to clients. M is the number of changed blocks. |

## Integration Patterns

### Standard Usage
These systems are not invoked directly. They are registered with the world's component store at startup and are automatically executed by the ECS engine when their respective queries and dependencies are met.

```java
// Example of system registration during server setup
ChunkStore chunkStore = world.getChunkStore();

// The engine discovers and registers systems like these
chunkStore.addSystem(new ChunkSystems.OnNewChunk());
chunkStore.addSystem(new ChunkSystems.OnChunkLoad());
// ... and so on for all other systems.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Never create an instance of a system and call its methods (e.g., `tick`, `onEntityAdd`). The ECS engine manages the lifecycle and invocation context, providing necessary arguments like the CommandBuffer. Bypassing the engine will lead to state corruption and concurrency violations.
-   **Incorrect Dependency Ordering:** The systems rely on a specific execution order, managed by the `Dependency` API. For example, OnChunkLoad must run *after* OnNewChunk. Modifying this order during registration will break the chunk initialization pipeline.

## Data Pipeline

### Chunk Initialization Pipeline
This flow describes how an empty chunk entity is progressively built into a fully interactive part of the world.

> Flow:
> 1.  **World Generator** creates an entity with a **WorldChunk** component.
> 2.  **OnNewChunk** system detects this entity.
> 3.  `onEntityAdd` fires, adding a **ChunkColumn** component to the entity.
> 4.  **OnChunkLoad** system now detects the entity (which has both WorldChunk and ChunkColumn).
> 5.  `onEntityAdded` fires, creating multiple child entities, each with a **ChunkSection** component.
> 6.  **EnsureBlockSection** system detects each new ChunkSection entity.
> 7.  `onEntityAdd` fires, adding a **BlockSection** component to each section.
> 8.  The chunk is now fully initialized and ready for block data population and interaction.

### Block Change Replication Pipeline
This flow describes how a single block change on the server is broadcast to clients.

> Flow:
> 1.  Game logic (e.g., player action) calls `BlockSection.setBlock(pos, id)`.
> 2.  The `setBlock` method records the position index in an internal `IntOpenHashSet` of changes.
> 3.  On the next server tick, the **ReplicateChanges** system's `tick` method is called for this BlockSection.
> 4.  The system retrieves and clears the set of changed positions.
> 5.  It queries the world for players whose **ChunkTracker** is monitoring this chunk.
> 6.  It constructs an optimized network packet (**ServerSetBlock**, **ServerSetBlocks**, or **SetChunk**) based on the number of changes.
> 7.  The packet is written to the packet handler for each relevant player.
> 8.  **Network Layer** sends the packet to the client.

