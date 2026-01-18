---
description: Architectural reference for RefChangeSystem
---

# RefChangeSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract Base Class / Template

## Definition
```java
// Signature
public abstract class RefChangeSystem<ECS_TYPE, T extends Component<ECS_TYPE>> 
    extends System<ECS_TYPE> 
    implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
RefChangeSystem is a foundational, reactive component within the Hytale Entity-Component-System (ECS) framework. It provides a robust, event-driven template for executing logic in response to lifecycle changes of a specific component type.

Instead of imperatively polling for state changes across thousands of entities each frame, developers implement subclasses of RefChangeSystem to be notified automatically when a component of interest is added, modified, or removed. This inverts control and promotes a highly decoupled and efficient architecture.

The system acts as a specialized observer, where the ECS World is the subject. When an operation like `store.add(entity, component)` is performed, the World dispatches events to all registered RefChangeSystem implementations that are tracking that specific component type.

A critical design feature is the mandatory use of the CommandBuffer for subsequent ECS mutations within the callbacks. This enforces a deferred execution model, queuing all resulting state changes to be applied atomically at the end of the current tick. This pattern is essential for preventing non-deterministic behavior and concurrent modification exceptions during system processing.

## Lifecycle & Ownership
-   **Creation:** Concrete implementations of RefChangeSystem are not instantiated directly by user code. The ECS World's SystemManager discovers and instantiates all system implementations during the world's initialization phase, typically via reflection or a predefined registry.
-   **Scope:** A RefChangeSystem instance has a lifecycle tightly coupled to its parent ECS World. It persists for the entire duration of the world, such as a client game session or a server instance.
-   **Destruction:** The system is marked for garbage collection when its parent ECS World is destroyed. No manual cleanup is required by implementers unless they manage external, unmanaged resources.

## Internal State & Concurrency
-   **State:** The RefChangeSystem base class is entirely stateless. However, concrete subclasses are free to introduce mutable state to track information across multiple ticks or entities.

    **Warning:** Any state stored in a subclass must be managed carefully, as the system instance is long-lived. Avoid accumulating data that could lead to memory leaks.

-   **Thread Safety:** The Hytale ECS framework guarantees that all callback methods (`onComponentAdded`, `onComponentSet`, `onComponentRemoved`) are invoked from a single, dedicated game-tick thread. Therefore, logic and state manipulation *within* a single system instance are inherently thread-safe with respect to other systems.

    However, if the system's internal state is accessed by other threads (e.g., a rendering thread reading position data), that access **must be explicitly synchronized** by the developer. The CommandBuffer is the only safe way to propagate changes back into the ECS World from within a callback.

## API Surface
The public API of RefChangeSystem is a contract to be fulfilled by subclasses, not a set of methods to be called by external code. The ECS framework invokes these methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| componentType() | ComponentType | O(1) | **Implementation Required.** Must return the type token of the component this system tracks. |
| onComponentAdded(ref, component, store, commands) | void | User-Defined | **Callback.** Invoked when the tracked component is first added to an entity. |
| onComponentSet(ref, old, new, store, commands) | void | User-Defined | **Callback.** Invoked when an existing component on an entity is replaced with a new instance. |
| onComponentRemoved(ref, component, store, commands) | void | User-Defined | **Callback.** Invoked when the tracked component is removed from an entity. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a new class that extends RefChangeSystem, specifying the entity type and the component to observe. This is ideal for logic that must react to an entity gaining or losing a capability, such as a `PlayerComponent` or `PhysicsComponent`.

```java
// A system to manage particle effects when a "Burning" status is applied or removed.
public class BurningEffectSystem extends RefChangeSystem<World, BurningComponent> {

    @Override
    @Nonnull
    public ComponentType<World, BurningComponent> componentType() {
        // This system is only interested in BurningComponent changes.
        return BurningComponent.TYPE;
    }

    @Override
    public void onComponentAdded(@Nonnull Ref<World> entity, @Nonnull BurningComponent burning, @Nonnull Store<World> store, @Nonnull CommandBuffer<World> commands) {
        // An entity started burning. Spawn a particle effect.
        commands.add(entity, new ParticleEffectComponent("fire.particle"));
    }

    @Override
    public void onComponentSet(@Nonnull Ref<World> entity, @Nullable BurningComponent old, @Nonnull BurningComponent new, @Nonnull Store<World> store, @Nonnull CommandBuffer<World> commands) {
        // The burning effect was modified (e.g., duration extended).
        // We could potentially modify the particle effect here if needed.
    }

    @Override
    public void onComponentRemoved(@Nonnull Ref<World> entity, @Nonnull BurningComponent burning, @Nonnull Store<World> store, @Nonnull CommandBuffer<World> commands) {
        // An entity is no longer burning. Remove the associated particle effect.
        commands.remove(entity, ParticleEffectComponent.TYPE);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new MyChangeSystem()`. The engine's SystemManager is solely responsible for the lifecycle of systems. Direct instantiation will result in a non-functional system that never receives events.
-   **Direct State Mutation:** Do not modify the `Store` object directly from within a callback. This will break the deterministic execution of the game tick and likely throw a `ConcurrentModificationException`. Always use the provided `CommandBuffer` to queue changes.
-   **Blocking Operations:** Callbacks must execute quickly. Performing blocking operations like file I/O, database queries, or long-running computations will stall the entire game tick, causing severe performance degradation. Offload such work to separate worker threads.

## Data Pipeline
RefChangeSystem operates as a reactive sink in the core data modification pipeline. It does not transform data but rather triggers side effects in response to data changes.

> Flow:
> External Action (e.g., Player Input, Network Event) -> `Store.add(entity, component)` -> ECS World Event Dispatcher -> **`RefChangeSystem.onComponentAdded(...)`** -> `CommandBuffer.add(...)` -> End-of-Tick Command Processor -> Finalized World State

