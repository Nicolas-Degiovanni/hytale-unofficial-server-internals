---
description: Architectural reference for EntityContainer
---

# EntityContainer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.views
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface EntityContainer {
   void addEntity(@Nonnull EntityPlacementData var1);

   boolean isInsideBuffer(int var1, int var2, int var3);
}
```

## Architecture & Concepts
The EntityContainer interface defines a contract for transient, write-only data structures used during world generation. It serves as a temporary buffer to collect entity placement information for a specific, localized generation unit, such as a world chunk or region.

This interface acts as a critical decoupling layer. It allows high-level procedural generation systems (e.g., BiomePopulators, StructureGenerators) to declare their intent to place an entity without needing knowledge of the underlying chunk or region data storage. The generator simply interacts with the EntityContainer contract, while the world generation orchestrator provides the concrete implementation, managing the final commit of this data to the world.

The primary responsibilities of an implementing class are:
1.  To accumulate `EntityPlacementData` objects during a single, atomic generation pass.
2.  To provide a spatial query mechanism to check if a given coordinate falls within the container's operational bounds.

## Lifecycle & Ownership
As an interface, EntityContainer has no lifecycle itself. The following pertains to any concrete implementation provided by the generation framework.

-   **Creation:** An implementation of EntityContainer is instantiated by the world generation orchestrator at the beginning of a discrete generation task (e.g., the population phase for a single chunk). It is never created directly by feature or biome populators.
-   **Scope:** The object's lifetime is strictly limited to the duration of the generation task it was created for. It is a short-lived, transient object.
-   **Destruction:** Upon completion of the generation pass, the orchestrator consumes the data from the EntityContainer and flushes it to the persistent chunk data store. The container object is then discarded and becomes eligible for garbage collection. Holding a reference to an EntityContainer beyond this point is a critical error.

## Internal State & Concurrency
-   **State:** Any implementation of this interface is inherently stateful and mutable. Its core purpose is to aggregate a collection of EntityPlacementData objects via the `addEntity` method. The internal state is effectively a write-log of entities to be placed.
-   **Thread Safety:** This interface makes no guarantees of thread safety. Implementations are expected to be confined to a single worker thread. Sharing a single EntityContainer instance across multiple threads without explicit, external synchronization will lead to race conditions and world corruption. The standard pattern is one EntityContainer instance per generation worker thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntity(EntityPlacementData) | void | O(1) | Appends an entity placement request to the internal buffer. This is a fire-and-forget operation from the caller's perspective. |
| isInsideBuffer(int, int, int) | boolean | O(1) | Checks if the specified world coordinates are within the spatial bounds managed by this container. Does not check for entity collisions. |

## Integration Patterns

### Standard Usage
The canonical usage involves a generator receiving an EntityContainer from a higher-level system, using it to stage entities, and then relinquishing control.

```java
// A generator receives the container as part of its execution context
public void populate(WorldGenContext context, EntityContainer entityBuffer) {
    // Decide to place a tree
    if (shouldPlaceTreeAt(context.getPos())) {
        EntityPlacementData treeData = createTreeEntityData(context.getPos());
        entityBuffer.addEntity(treeData);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Reference Hoarding:** Do not store a reference to an EntityContainer in a long-lived object. It is a transient buffer, and its contents are invalid after the generation pass completes.
-   **Cross-Thread Sharing:** Never pass an EntityContainer from one generation thread to another. Each thread processing a world unit must be provided with its own dedicated container instance by the orchestrator.
-   **Manual Implementation:** Do not implement this interface in client code. The world generation framework provides canonical implementations that are tightly integrated with the engine's data structures.

## Data Pipeline
EntityContainer acts as a staging area in the world generation data flow. It buffers commands from procedural logic before they are committed to the final world state.

> Flow:
> Procedural Logic (e.g., BiomePopulator) -> Creates EntityPlacementData -> **EntityContainer**.addEntity() -> (Pass Complete) -> World Generation Orchestrator -> Flushes Buffer to Chunk Entity List

