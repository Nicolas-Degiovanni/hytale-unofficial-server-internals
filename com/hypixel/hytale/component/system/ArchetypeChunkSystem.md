---
description: Architectural reference for ArchetypeChunkSystem
---

# ArchetypeChunkSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class ArchetypeChunkSystem<ECS_TYPE> extends System<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
The ArchetypeChunkSystem is a foundational component of Hytale's high-performance, data-oriented Entity Component System (ECS) framework. It represents a specialized type of System designed to operate on entire blocks of entities, known as ArchetypeChunks, rather than processing entities one by one.

This design is a cornerstone of engine performance. An **Archetype** is a unique combination of component types. An **ArchetypeChunk** is a contiguous block of memory containing all entities that share the exact same Archetype. By processing entire chunks, systems can achieve massive performance gains through cache-friendly, linear memory access and the potential for SIMD (Single Instruction, Multiple Data) optimizations.

This class acts as a reactive listener. The engine's ECS World automatically notifies an ArchetypeChunkSystem whenever an ArchetypeChunk is created or destroyed that matches the query criteria defined by the system's implementation of the QuerySystem interface. This allows the system to manage its relationship with relevant data chunks without needing to poll or search for them.

## Lifecycle & Ownership
The lifecycle of an ArchetypeChunkSystem is distinct from the lifecycle of its *connection* to a specific ArchetypeChunk.

-   **Creation:** Concrete implementations of this class are instantiated once by the engine's central SystemManager during world initialization or a similar bootstrap phase. They are long-lived, foundational objects.
-   **Scope:** An instance of an ArchetypeChunkSystem persists for the entire lifetime of the game world it belongs to.
-   **Destruction:** The system is destroyed and cleaned up only when the game world is fully unloaded. The `onSystemRemovedFromArchetypeChunk` method is not for the system's destruction, but for severing the link to a chunk that is being destroyed or no longer matches the system's query.

## Internal State & Concurrency
-   **State:** This abstract base class is stateless. However, concrete implementations are expected to be stateful. They may cache references to ArchetypeChunks or maintain aggregate data derived from them.
-   **Thread Safety:** **WARNING:** Implementations of this class must be designed with concurrency in mind. While the `onSystemAdded...` and `onSystemRemoved...` callbacks are typically invoked from a main thread or under a world lock, the primary processing logic of a System is often executed in parallel across multiple worker threads. Any shared state within a subclass must be protected by appropriate synchronization mechanisms to prevent race conditions.

## API Surface
The public contract consists of two lifecycle callbacks that subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSystemAddedToArchetypeChunk(chunk) | void | O(N) | Callback invoked by the ECS World when a new ArchetypeChunk matching the system's query is created. |
| onSystemRemovedFromArchetypeChunk(chunk) | void | O(N) | Callback invoked by the ECS World when an ArchetypeChunk is destroyed or no longer matches the system's query. |

## Integration Patterns

### Standard Usage
A developer does not call methods on this class directly. Instead, they extend it to create a new system that reacts to the existence of specific entity compositions.

```java
// Example: A system that processes all entities with Position and Velocity
public class MovementSystem extends ArchetypeChunkSystem<EcsType> {

    private List<ArchetypeChunk<EcsType>> relevantChunks = new ArrayList<>();

    // The engine calls this when a chunk with Position and Velocity is created
    @Override
    public void onSystemAddedToArchetypeChunk(ArchetypeChunk<EcsType> chunk) {
        this.relevantChunks.add(chunk);
    }

    // The engine calls this when the chunk is destroyed
    @Override
    public void onSystemRemovedFromArchetypeChunk(ArchetypeChunk<EcsType> chunk) {
        this.relevantChunks.remove(chunk);
    }

    // The main update loop would then iterate over relevantChunks
    public void update(float deltaTime) {
        // ... process all entities within the cached chunks
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call `onSystemAddedToArchetypeChunk` or `onSystemRemovedFromArchetypeChunk` manually. These are strictly managed by the engine's ECS World to guarantee state consistency.
-   **Blocking Operations:** Do not perform file I/O, network requests, or other long-running, blocking operations inside these callbacks. They are expected to execute quickly to avoid stalling the main world update tick.
-   **Unsafe State Management:** Do not store references to ArchetypeChunks without removing them in `onSystemRemovedFromArchetypeChunk`. This will lead to memory leaks and attempts to access deallocated memory.

## Data Pipeline
This system does not process a continuous stream of data in a traditional sense. Instead, it reacts to structural changes within the ECS World itself.

> Flow:
> Entity Created/Destroyed -> ECS World detects Archetype change -> A new ArchetypeChunk is created or an old one is destroyed -> QuerySystem Matcher -> **ArchetypeChunkSystem** callback is invoked -> System updates its internal list of relevant chunks

