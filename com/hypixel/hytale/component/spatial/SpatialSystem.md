---
description: Architectural reference for SpatialSystem
---

# SpatialSystem

**Package:** com.hypixel.hytale.component.spatial
**Type:** Abstract System

## Definition
```java
// Signature
public abstract class SpatialSystem<ECS_TYPE> extends TickingSystem<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
The SpatialSystem is an abstract base class that forms a critical part of the engine's spatial query infrastructure within the Entity-Component-System (ECS) framework. Its sole responsibility is to populate and rebuild a spatial partitioning data structure (e.g., an octree, k-d tree, or spatial hash grid) every game tick. This allows other systems to perform efficient spatial queries, such as "find all entities within a 10-meter radius".

As a subclass of TickingSystem, it is automatically integrated into the main game loop. The core architectural choice is a **full rebuild strategy**. On each invocation of its `tick` method, the system completely discards the previous frame's spatial data and reconstructs it by iterating over all relevant entities. This design favors simplicity and robustness in a highly dynamic environment over the performance gains of incremental updates, thereby avoiding complex state synchronization and cache invalidation problems.

This class is abstract and cannot be used directly. Engine developers must extend it to create concrete implementations for specific entity types. The primary extension point is the `getPosition` method, which defines the logic for extracting a world position from a given entity's component data.

## Lifecycle & Ownership
-   **Creation:** Concrete subclasses of SpatialSystem are instantiated by the core engine during the system registration phase, typically when a world or game session is initialized. They are configured with a specific `ResourceType` that links them to the `SpatialResource` they are responsible for managing.
-   **Scope:** An instance of a SpatialSystem persists for the entire lifetime of the ECS `Store` it operates on. This generally corresponds to the duration of a game world, from loading to unloading.
-   **Destruction:** The system is deregistered and becomes eligible for garbage collection when its parent `Store` is destroyed. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** A SpatialSystem instance is effectively stateless. Its only internal field is a final reference to a `ResourceType`. The state it manipulates—the spatial data itself—is owned by a `SpatialResource` object managed by the central ECS `Store`. The `tick` process is a destructive and reconstructive operation on this external state.
-   **Thread Safety:** This system is **not thread-safe**. The `tick` method is designed to be executed by a single, dedicated ECS thread. Calling `tick` or manipulating the associated `SpatialResource` from multiple threads will result in race conditions, data corruption, and unpredictable engine crashes. All queries against the spatial structure must be synchronized with the ECS tick cycle, typically by performing them from other systems that are scheduled to run after the relevant SpatialSystem.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | Executes one update cycle. Clears, populates, and rebuilds the associated spatial structure. N is the number of entities processed. Called exclusively by the ECS scheduler. |
| getPosition(chunk, index) | Vector3d | O(1) | **Abstract.** Concrete implementations must provide the logic to extract a position vector from an entity's components. |

## Integration Patterns

### Standard Usage
The correct pattern is to extend SpatialSystem and implement the `getPosition` method. This new class is then registered with the engine's system scheduler.

```java
// A concrete implementation for entities that have a PositionComponent.
public class MobileEntitySpatialSystem extends SpatialSystem<ServerWorld> {

    // The system is provided with the resource it will manage.
    public MobileEntitySpatialSystem(ResourceType<ServerWorld, SpatialResource<Ref<ServerWorld>, ServerWorld>> resourceType) {
        super(resourceType);
    }

    @Override
    @Nullable
    public Vector3d getPosition(@Nonnull ArchetypeChunk<ServerWorld> chunk, int index) {
        // This logic is specific to this system. It knows that its target
        // entities are guaranteed to have a PositionComponent.
        PositionComponent pos = chunk.getComponent(PositionComponent.class, index);
        if (pos == null) {
            // This entity does not have a position and will be ignored.
            return null;
        }
        return pos.getCoordinates();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MobileEntitySpatialSystem()` in general game logic. Systems are part of the core engine infrastructure and must be registered with the ECS scheduler to function correctly.
-   **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the engine's scheduler, breaks system ordering guarantees, and can introduce severe threading issues.
-   **Stateful Implementation:** The `getPosition` method should be a pure function of its inputs (`chunk`, `index`). Avoid reading from or writing to external state within this method, as it is called in a tight loop and must be highly performant and free of side effects.

## Data Pipeline
The system transforms raw component data from the ECS store into an optimized spatial data structure ready for querying.

> Flow:
> ECS Store (All Entity Chunks) -> **SpatialSystem.tick()** iterates entities -> **getPosition()** extracts Vector3d -> Intermediate `SpatialData` buffer -> `SpatialStructure.rebuild()` -> Optimized Spatial Structure (e.g., Octree)

