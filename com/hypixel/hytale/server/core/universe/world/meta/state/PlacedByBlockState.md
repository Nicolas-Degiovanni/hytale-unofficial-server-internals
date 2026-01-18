---
description: Architectural reference for PlacedByBlockState
---

# PlacedByBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface PlacedByBlockState {
   void placedBy(@Nonnull Ref<EntityStore> var1, @Nonnull String var2, @Nonnull BlockState var3, @Nonnull ComponentAccessor<EntityStore> var4);
}
```

## Architecture & Concepts
The PlacedByBlockState interface defines a behavioral contract for server-side game objects, primarily blocks, that need to react to being placed in the world. It serves as a specialized callback mechanism, deeply integrated with the server's Entity Component System (ECS).

This interface embodies the **Strategy Pattern**. Instead of embedding complex placement logic directly into the core world modification systems, this logic is encapsulated within objects that implement this interface. The world engine can then invoke this behavior polymorphically on any block that possesses it.

Its primary role is to decouple the generic act of "placing a block" from the specific, stateful consequences of that action. For example, a standard stone block might have no placement logic, while a custom "turret" block could implement this interface to immediately spawn an associated turret entity, orient itself towards the placer, and register itself with a defense system. The arguments passed to the `placedBy` method provide the full context of the placement event, including a reference to the entity that performed the action.

## Lifecycle & Ownership
As an interface, PlacedByBlockState does not have a lifecycle of its own. Instead, it defines a contract that other concrete classes implement. The lifecycle of an object implementing this interface is determined by its container.

- **Creation:** An object implementing this interface is typically instantiated as part of a larger component or block definition. This often occurs at server startup when game assets and behaviors are loaded and registered.
- **Scope:** The scope is tied to the lifetime of the implementing object. If it is part of a block's definition, it will exist for the entire server session.
- **Destruction:** The implementing object is destroyed when its owner is destroyed. For block behaviors, this typically happens during server shutdown.

**WARNING:** The `placedBy` method is a callback. Its execution is ephemeral and tied to the specific moment a block is placed in the world. It does not persist or have a lifecycle beyond its immediate invocation.

## Internal State & Concurrency
- **State:** This interface is stateless. However, implementations are expected to be highly stateful, modifying the world state based on the placement event.
- **Thread Safety:** **Not guaranteed.** Implementations of this interface are responsible for their own thread safety. The `placedBy` method is invoked by the server's world modification thread. Any long-running operations or asynchronous tasks initiated within an implementation must use proper synchronization primitives to avoid corrupting world state.

**WARNING:** Implementations of `placedBy` are executed on a critical server path. Blocking operations will directly contribute to server latency and should be avoided. Any modifications to the passed `EntityStore` or world via the `ComponentAccessor` must be considered atomic or be protected by appropriate locks if they involve multi-step transactions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| placedBy(placerStore, placerName, blockState, accessor) | void | Implementation-Dependent | Callback invoked by the world engine when a block with this behavior is placed. Provides context about the placer and the block itself. |

## Integration Patterns

### Standard Usage
A developer will implement this interface on a component that defines a block's unique server-side behavior. This component is then attached to a block's definition. The engine handles the invocation automatically.

```java
// Hypothetical implementation for a block that spawns a light source
public class MagicCrystalBehavior implements PlacedByBlockState {

    @Override
    public void placedBy(@Nonnull Ref<EntityStore> placerStore, @Nonnull String placerName, @Nonnull BlockState placedBlock, @Nonnull ComponentAccessor<EntityStore> worldAccessor) {
        // Get the position from the BlockState
        Vector3i position = placedBlock.getPosition();

        // Use the accessor to interact with the world's ECS
        EntityStore world = worldAccessor.getStore();
        world.createEntity(position.add(0, 1, 0), "hytale:light_source_entity");

        // Log the placement event
        System.out.println("Magic Crystal placed by " + placerName + " at " + position);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the `placedBy` method directly from your own code. It is a system-level callback and invoking it manually will bypass engine-level state management, leading to world corruption or desynchronization.
- **Blocking Operations:** Do not perform file I/O, database queries, or long-running computations within the `placedBy` method. This will stall the world thread and cause severe server lag. Offload heavy work to a separate worker thread pool.
- **Assuming Thread Affinity:** Do not assume this method will always be called from the same thread, especially in future engine versions that may use multi-threaded chunk processing. Write thread-safe and re-entrant code.

## Data Pipeline
This component acts as a sink in a control-flow pipeline, translating a world event into a specific game-logic action.

> Flow:
> Player Action -> Network Packet -> Server World Ticker -> Block Placement Logic -> **PlacedByBlockState.placedBy()** -> World State Mutation (e.g., Entity Spawn, Component Update)

