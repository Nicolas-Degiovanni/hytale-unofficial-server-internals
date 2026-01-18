---
description: Architectural reference for ArchetypeChunk
---

# ArchetypeChunk

**Package:** com.hypixel.hytale.component
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ArchetypeChunk<ECS_TYPE> {
```

## Architecture & Concepts
The ArchetypeChunk is the fundamental unit of data storage within Hytale's Entity-Component-System (ECS) architecture. It is a memory-optimized container designed to hold a collection of entities that share the exact same set of components, defined by an associated **Archetype**.

The primary design goal of this class is to achieve high-performance data access through memory locality. It implements a **Structure of Arrays (SoA)** data layout. Instead of storing all components for a single entity together (Array of Structures), it stores all components of a single type together. For example, all Position components for every entity in the chunk are packed into a contiguous array. This layout is extremely cache-friendly for systems that iterate over large numbers of entities to process a specific subset of their components, which is the dominant access pattern in an ECS.

An ArchetypeChunk is not a standalone object; it is a managed "bucket" within the higher-level **Store** service. The Store is responsible for routing entities to the correct chunk based on their archetype. When an entity's component composition changes, the Store orchestrates its migration from its old ArchetypeChunk to a new one that matches its new archetype.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the **Store** service. A new ArchetypeChunk is created on-demand when an entity is assigned to an Archetype that either has no existing chunks or whose existing chunks are at full capacity.
- **Scope:** The lifecycle of an ArchetypeChunk is intrinsically linked to the entities it contains. It persists as long as at least one entity matching its Archetype exists within it.
- **Destruction:** An ArchetypeChunk becomes eligible for garbage collection once its entity count drops to zero. This occurs when all entities are either destroyed or transferred to other chunks. The Store is responsible for removing its reference to an empty chunk, allowing the memory to be reclaimed.

## Internal State & Concurrency
- **State:** The ArchetypeChunk is a highly mutable, stateful data container. Its core fields, including **components**, **refs**, and **entitiesSize**, are constantly modified by entity addition, removal, and transfer operations. It directly owns the component data for the entities it manages.

- **Thread Safety:** **This class is not thread-safe.** All public methods perform direct, unsynchronized mutations on internal arrays. Concurrent access from multiple threads will lead to race conditions, data corruption, and catastrophic system instability. All interactions with an ArchetypeChunk instance must be externally synchronized, typically by confining its usage to a single, main game-loop thread or a job system that guarantees exclusive write access.

## API Surface
The public API is designed for high-frequency, low-level data manipulation by the parent Store service.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntity(ref, holder) | int | O(1) amortized | Adds a new entity and its components. Complexity becomes O(N*M) during array resize, where N is the new capacity and M is the number of component types. |
| removeEntity(index, target) | Holder | O(M) | Removes an entity using an efficient swap-and-pop algorithm. Complexity is proportional to the number of component types (M) being copied. |
| getComponent(index, type) | T | O(1) | Retrieves a specific component for a specific entity via direct array lookup. |
| setComponent(index, type, component) | void | O(1) | Overwrites a specific component for a specific entity via direct array write. |
| transferTo(...) | void | O(N*M) | High-cost operation to migrate all entities from this chunk to another. Used when an Archetype changes structurally. |

## Integration Patterns

### Standard Usage
Direct interaction with ArchetypeChunk is not a standard developer workflow. All operations are abstracted away by higher-level managers. Systems query for entities and receive iterable collections, which internally access one or more ArchetypeChunks to provide the component data.

```java
// A System's perspective (conceptual)
// The system does NOT see the ArchetypeChunk directly.
// It gets an iterator or view managed by the Store.

for (Entity entity : world.query(Position.class, Velocity.class)) {
    Position pos = entity.getComponent(Position.class);
    Velocity vel = entity.getComponent(Velocity.class);
    
    // ... process components ...
    // Under the hood, this loop is iterating through one or more
    // ArchetypeChunks that match the (Position, Velocity) archetype.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ArchetypeChunk()`. The ECS **Store** is the sole owner and manager of chunk lifecycles. Manual creation will break the internal state of the entity database.
- **Unsynchronized Access:** Accessing a chunk from multiple threads without external locking will corrupt memory. The ECS assumes a single-threaded writer model for its core data structures.
- **Stale References:** Do not cache a reference to an ArchetypeChunk. Entities are frequently moved between chunks. A cached reference can quickly become stale, pointing to a chunk that no longer contains the entity you are interested in, or may have been deallocated entirely.

## Data Pipeline
The ArchetypeChunk is a destination and source for component data, orchestrated by the Store.

**Entity Archetype Change Pipeline:**
> Flow:
> EntityManager.addComponent() -> **Store** calculates new Archetype -> **Store** finds or creates target ArchetypeChunk -> OldChunk.removeEntity() -> **NewChunk.addEntity()**

**System Query Pipeline:**
> Flow:
> System requests entities -> **Store** identifies all matching ArchetypeChunks -> System iterates through each **ArchetypeChunk** -> System calls **chunk.getComponent()** to read/write data.

