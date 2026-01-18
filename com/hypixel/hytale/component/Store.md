---
description: Architectural reference for Store
---

# Store<ECS_TYPE>

**Package:** com.hypixel.hytale.component
**Type:** Managed Instance

## Definition
```java
// Signature
public class Store<ECS_TYPE> implements ComponentAccessor<ECS_TYPE> {
```

## Architecture & Concepts
The Store is the heart of Hytale's Entity Component System (ECS). It is the primary container for all game state within a specific context, such as a client world or a server instance. It is not a global singleton; multiple Store instances can exist, each representing an isolated simulation.

Conceptually, a Store is analogous to a "World" in other game engines. It manages the entire lifecycle of entities, the components attached to them, and the systems that operate on that data.

The core architectural pattern is **Data-Oriented Design (DOD)**, centered around the concept of **Archetypes**. An Archetype represents a unique combination of component types. All entities sharing the exact same set of components belong to the same Archetype.

Entities of the same Archetype are stored contiguously in memory within an **ArchetypeChunk**. This layout ensures that when a System queries for entities with specific components, the required data is packed tightly together, leading to highly efficient, cache-friendly iteration. This is the fundamental performance characteristic of the entire ECS.

Key responsibilities of the Store include:
-   **State Management:** Owns all entities, components, and singleton-like Resources.
-   **Structural Changes:** Provides a transactional API for creating, deleting, and modifying entities and their components. These operations often involve moving entity data between different ArchetypeChunks.
-   **System Execution:** Orchestrates the execution of Systems, including ticking, event handling, and data queries.
-   **Thread Confinement:** Enforces strict single-threaded access to its state. All mutations and reads must occur on the thread that created the Store.
-   **Command Buffering:** Utilizes a CommandBuffer pattern to safely queue structural changes that occur during system processing. This prevents iterator invalidation and ensures a deterministic order of operations.

The generic parameter ECS_TYPE allows the Store to be associated with external, context-specific data (e.g., client-only rendering data) without polluting the core ECS logic.

### Lifecycle & Ownership
-   **Creation:** A Store is never instantiated directly by application code. Its constructor is package-private. It is created and managed exclusively by a **ComponentRegistry**, which acts as the factory for the entire ECS instance.
-   **Scope:** The lifetime of a Store is tied to a specific game session or world. For example, a client-side Store is created when a player joins a world and is destroyed when they disconnect. It persists for the entire duration of that session.
-   **Destruction:** The public `shutdown()` method initiates a graceful teardown. This is a critical, irreversible process that:
    1.  Notifies all registered StoreSystem instances of their removal.
    2.  Saves all serializable Resource data via the IResourceStorage interface.
    3.  Invalidates all outstanding entity references (Ref).
    4.  Releases all internal memory, including ArchetypeChunks and entity lookup tables.

**WARNING:** Failure to call `shutdown()` will result in memory leaks and prevent proper resource persistence.

## Internal State & Concurrency
-   **State:** The Store is a highly stateful and mutable class. Its primary state consists of several interconnected data structures that enable efficient entity management:
    -   **archetypeChunks:** An array of ArchetypeChunk objects, which contain the raw, contiguous component data.
    -   **refs:** A sparse array mapping an entity's integer index to its stable Ref handle.
    -   **entityToArchetypeChunk / entityChunkIndex:** Sparse arrays that form an indirection layer. Given an entity index, these tables provide an O(1) lookup to find which ArchetypeChunk the entity resides in and its index within that chunk's dense arrays.
    -   **systemIndexToArchetypeChunkIndexes:** A critical performance cache. It is an array of BitSets mapping a System's index to the set of ArchetypeChunks it needs to process. This avoids iterating over every Archetype in the Store for every System.

-   **Thread Safety:** **The Store is NOT thread-safe and is strictly thread-confined.**
    -   Upon creation, it captures the current thread. Nearly every public method begins with `assertThread()`, which will throw an `IllegalStateException` if called from any other thread.
    -   The internal `ProcessingCounter` lock is **not a concurrency primitive**. It is a re-entrant counter used to prevent API misuse. It blocks direct structural changes (like `addEntity`) from within a System's execution path, forcing the use of a CommandBuffer. This is enforced by `assertWriteProcessing()`.
    -   Parallelism is explicitly managed for specific, high-level operations like `forEachEntityParallel`. In these cases, the Store partitions the work and provides thread-safe CommandBuffers, merging the results in a controlled manner.

**WARNING:** Any external access to a Store instance from a different thread without using the provided parallel APIs will lead to state corruption, race conditions, and undefined behavior.

## API Surface
The public API provides methods for entity manipulation, system execution, and resource access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntity(holder, reason) | Ref | Amortized O(C) | Creates a new entity. Involves finding or creating an ArchetypeChunk and copying initial component data. C is component count. |
| removeEntity(ref, reason) | Holder | O(C) | Destroys an entity. Uses a swap-and-pop on the entity list. Copies component data to the returned Holder. |
| addComponent(ref, type, component) | void | O(C) | **Expensive.** Moves the entity to a new ArchetypeChunk, copying all of its existing components. |
| removeComponent(ref, type) | void | O(C) | **Expensive.** Moves the entity to a new ArchetypeChunk, copying all of its remaining components. |
| getComponent(ref, type) | T | O(1) | Highly efficient lookup via the entity index indirection tables. |
| getResource(type) | T | O(1) | Direct array lookup for a store-scoped singleton resource. |
| tick(dt) | void | O(S+E) | Primary entry point for the game loop. Iterates and executes all tickable systems. S is system count, E is total entities processed. |
| invoke(event) | void | O(S) | Dispatches a world or entity event to all listening systems. |

## Integration Patterns

### Standard Usage
A developer typically interacts with the Store through a context object or service locator. The primary interaction is either dispatching events and ticks from the main game loop or querying/mutating data from within a System.

```java
// Main game loop
void gameLoop(Store<ClientECS> clientStore, float deltaTime) {
    clientStore.tick(deltaTime);
}

// Inside a System
public class MySystem extends RefSystem<ClientECS> {
    public void onEntityAdded(Ref<ClientECS> ref, AddReason reason, ComponentAccessor<ClientECS> accessor, CommandBuffer<ClientECS> commands) {
        // Use the accessor (the Store) to get data
        Position pos = accessor.getComponent(ref, Position.TYPE);
        
        // Use the command buffer for structural changes
        if (pos.isInvalid()) {
            commands.removeEntity(ref, RemoveReason.INVALID);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new Store()`. The constructor is internal. Always acquire a Store from its managing `ComponentRegistry`.
-   **Cross-Thread Access:** Do not share a Store instance across threads or call its methods from any thread other than the one it was created on. This will trigger an exception and is fundamentally unsafe.
-   **Mutation During Iteration:** Do not call methods like `addComponent` or `removeEntity` directly from within a system's `tick` or event handler. This will throw an `IllegalStateException` due to the processing lock. All structural changes must be queued on the provided `CommandBuffer`.
-   **Storing Invalid Refs:** Do not hold references to a `Ref<ECS_TYPE>` object after its corresponding entity has been removed. The `ref.isValid()` method should be used to check its status before use.

## Data Pipeline

### Entity Structural Change (e.g., addComponent)
This flow illustrates the expensive nature of changing an entity's archetype.

> Flow:
> `addComponent(ref, ...)` -> Validate Thread & Processing State -> Lock Processing Counter -> **1. Copy & Remove from Old Chunk:** `fromArchetypeChunk.removeEntity(...)` copies all components into a temporary `Holder`. -> **2. Modify Holder:** The new component is added to the `Holder`. -> **3. Find or Create New Chunk:** `findOrCreateArchetypeChunk(...)` gets the chunk for the new archetype. -> **4. Add to New Chunk:** `toArchetypeChunk.addEntity(...)` copies all components from the `Holder` into the new chunk's contiguous arrays. -> **5. Update Lookups:** The entity's entry in `entityToArchetypeChunk` and `entityChunkIndex` is updated to point to its new location. -> **6. Notify Systems:** Relevant `RefChangeSystem`s are invoked. -> **7. Cleanup:** If the old chunk is now empty, it is removed. -> Unlock Processing Counter -> Consume CommandBuffer.

### System Tick Execution
This flow shows how systems efficiently iterate over relevant entities.

> Flow:
> `store.tick(dt)` -> Get read lock on `ComponentRegistry` data -> Iterate over all registered `TickingSystem`s -> For each system: `system.tick(dt, systemIndex, this)` -> **1. Get Relevant Chunks:** Use `systemIndexToArchetypeChunkIndexes[systemIndex]` to get a `BitSet` of all matching `ArchetypeChunk`s. -> **2. Iterate Chunks:** Loop only over the `ArchetypeChunk`s indicated by the `BitSet`. -> **3. Process Entities:** The system's logic is executed on the dense, contiguous component arrays within each chunk. -> Any structural changes are queued on a `CommandBuffer`. -> After all systems run, the main `CommandBuffer` is consumed.

