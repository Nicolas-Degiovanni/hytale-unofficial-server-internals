---
description: Architectural reference for IBlackboardView
---

# IBlackboardView

**Package:** com.hypixel.hytale.server.npc.blackboard.view
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IBlackboardView<View extends IBlackboardView<View>> {
```

## Architecture & Concepts
The IBlackboardView interface is a core contract within the server-side NPC Artificial Intelligence system. It formalizes the concept of a "Blackboard View," which is a cached, partial, or transformed representation of an NPC's state or its perception of the world. In AI, a Blackboard is a centralized, shared memory space for knowledge. This interface defines a *view* into that knowledge, optimized for frequent access by specific AI behaviors.

Its primary architectural role is to decouple AI logic from the raw, underlying Entity Component System (ECS) data, represented here by EntityStore. By providing a stable, cached view of the world, it prevents different AI behaviors from repeatedly querying and processing the same complex state from the ECS on every tick. This is a critical performance optimization pattern.

The recursive generic type parameter, `View extends IBlackboardView<View>`, is a common pattern for ensuring type safety in fluent or factory-style interfaces. It guarantees that methods like getUpdatedView return the concrete implementation type (e.g., a `PathfindingView`), not the generic IBlackboardView, eliminating the need for unsafe casting by the consumer.

## Lifecycle & Ownership
- **Creation:** Implementations of IBlackboardView are not created directly. They are instantiated and managed by higher-level AI systems, such as a Behavior Tree or a central Blackboard manager for a given NPCEntity. A view is typically created when an NPC's AI is initialized or when a specific high-level behavior becomes active.
- **Scope:** The lifetime of a view is tightly coupled to the NPCEntity it represents or the AI behavior that requires it. It persists as long as the data it caches is relevant. The presence of the onWorldRemoved method indicates its lifecycle is strictly bound to the world session.
- **Destruction:** The owning AI system is responsible for invoking the `cleanup` or `onWorldRemoved` methods. This occurs when the associated NPC is despawned, the AI behavior is terminated, or the entire world is unloaded from the server. Failure to call these methods may result in memory leaks.

## Internal State & Concurrency
- **State:** This is an interface and therefore stateless. However, all concrete implementations are expected to be **highly stateful**. Their entire purpose is to cache data derived from the EntityStore. The contract, particularly the isOutdated and getUpdatedView methods, revolves around managing this cached state.
- **Thread Safety:** Implementations are **not thread-safe** and must not be treated as such. All interactions with an IBlackboardView instance should be confined to the server's main game thread or the specific thread responsible for ticking the associated NPC's AI. Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. The underlying ComponentAccessor and Store objects are not guaranteed to be safe for concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isOutdated(Ref, Store) | boolean | O(1) | Performs a fast check to determine if the view's cached data is stale compared to the authoritative EntityStore. **Warning:** This method must be lightweight and must not perform heavy computations. |
| getUpdatedView(Ref, ComponentAccessor) | View | O(N) | Constructs and returns a new, updated instance of the view with fresh data from the EntityStore. This follows a copy-on-write or immutable update pattern. |
| initialiseEntity(Ref, NPCEntity) | void | O(1) | Binds the view to a specific NPCEntity instance. This is typically called once immediately after the view is created. |
| cleanup() | void | O(1) | Releases any resources held by the view. Must be called by the owner before the object is discarded. |
| onWorldRemoved() | void | O(1) | A specific lifecycle hook called by the world management system during world unloading to ensure complete resource cleanup. |

## Integration Patterns

### Standard Usage
The intended pattern is a check-then-update cycle, typically executed at the beginning of an AI behavior's update tick. This ensures the AI always operates on sufficiently fresh data without the performance cost of rebuilding the view on every single tick.

```java
// An AI behavior holds a reference to its required view
private PathfindingView pathfindingView;

// In the AI's update tick
public void onTick(NPCEntity self) {
    // The EntityStore is the source of truth, provided by the engine
    Ref<EntityStore> entityRef = self.getRef();
    Store<EntityStore> entityStore = world.getEntityStore();

    if (this.pathfindingView.isOutdated(entityRef, entityStore)) {
        // The view is stale, so we request a new, updated one
        this.pathfindingView = this.pathfindingView.getUpdatedView(entityRef, entityStore);
    }

    // Proceed with AI logic using the now-guaranteed-fresh pathfindingView
    pathfindingView.calculatePath();
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Staleness:** Failing to call isOutdated and continuing to use a stale view can lead to NPCs making decisions based on old information, causing incorrect or bizarre behavior.
- **Constant Updates:** Calling getUpdatedView on every tick without checking isOutdated first. This completely negates the performance benefit of the caching pattern and may be less efficient than querying the EntityStore directly.
- **State Mutation:** Concrete implementations should avoid exposing methods that mutate their internal state. The correct pattern is to replace the entire view object with a new one via getUpdatedView.

## Data Pipeline
The IBlackboardView serves as a caching and transformation layer between the raw entity data and the AI decision-making logic.

> Flow:
> Authoritative State in **EntityStore** -> `getUpdatedView()` populates a concrete view implementation -> **IBlackboardView Instance** (Cached, Transformed State) -> AI Behavior (e.g., Pathfinding, Combat) -> NPC Action Execution

