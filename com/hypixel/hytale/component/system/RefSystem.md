---
description: Architectural reference for RefSystem
---

# RefSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class RefSystem<ECS_TYPE> extends System<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
The RefSystem is a foundational, event-driven building block within the Hytale Entity-Component-System (ECS) framework. It provides a reactive contract for systems that need to respond to the lifecycle of specific entities, rather than iterating over all entities every frame.

This class embodies the "System" in ECS, but with a specific focus on reaction to state changes. Concrete implementations of RefSystem are associated with a component query defined elsewhere. The ECS engine's query processor is responsible for monitoring the world state. When an entity's components change in a way that causes it to either match or cease to match a system's query, the engine automatically invokes the appropriate callback on that system.

This design pattern is highly efficient, as it avoids expensive, continuous polling. Systems only perform work when a relevant change has actually occurred. The inclusion of a CommandBuffer in the callback signatures is a critical architectural feature, enabling deferred and thread-safe modifications to the world state.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of RefSystem are discovered and instantiated by the core engine, typically during the world-loading or session initialization phase. They are registered with a central system manager or graph. Developers do not manually instantiate these systems.
- **Scope:** A RefSystem instance lives for the duration of the world or session it is registered with. Its lifecycle is directly managed by the parent ECS container.
- **Destruction:** The system is destroyed and its resources are released when the world is unloaded or the game session terminates.

## Internal State & Concurrency
- **State:** As an abstract base class, RefSystem is stateless. However, concrete implementations are expected to be stateful. They may cache entity references, manage handles to external resources (like physics bodies or audio sources), or maintain internal data structures required for their logic.
- **Thread Safety:** This class is **not** inherently thread-safe, and its contract relies on a strict execution model. The callbacks, onEntityAdded and onEntityRemove, are invoked by the ECS scheduler, which typically operates on a single, dedicated update thread.

    **WARNING:** Direct modification of the component Store from within these callbacks is an unsafe operation and can lead to data corruption or race conditions. All world-state mutations **must** be enqueued through the provided CommandBuffer. This ensures that all changes are applied deterministically at a designated synchronization point in the main game loop.

## API Surface
The public contract consists of two abstract methods that must be implemented by all subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(Ref, AddReason, Store, CommandBuffer) | void | O(N) | Callback invoked by the ECS engine when an entity begins to match the system's query. Complexity is dependent on subclass implementation. |
| onEntityRemove(Ref, RemoveReason, Store, CommandBuffer) | void | O(N) | Callback invoked by the ECS engine when an entity ceases to match the system's query. Complexity is dependent on subclass implementation. |

## Integration Patterns

### Standard Usage
A developer extends RefSystem to create logic that reacts to the appearance and disappearance of specific entity archetypes. The primary use case is for initialization and cleanup logic.

```java
// A system that creates and destroys a physics representation for entities
// that have both a Position and a Collider component.
public class PhysicsLifecycleSystem extends RefSystem<GameWorld> {

    private final PhysicsEngine physicsEngine;

    // Constructor for dependency injection
    public PhysicsLifecycleSystem(PhysicsEngine engine) {
        this.physicsEngine = engine;
    }

    @Override
    public void onEntityAdded(Ref<GameWorld> entity, AddReason reason, Store<GameWorld> store, CommandBuffer<GameWorld> commands) {
        // An entity with the required components has been added.
        // Create a corresponding physics body.
        PhysicsBody body = physicsEngine.createBodyFor(entity, store);

        // Add a new component to the entity to hold the physics body handle.
        // This is a deferred operation and is safe.
        commands.addComponent(entity, new PhysicsBodyComponent(body.getHandle()));
    }

    @Override
    public void onEntityRemove(Ref<GameWorld> entity, RemoveReason reason, Store<GameWorld> store, CommandBuffer<GameWorld> commands) {
        // The entity no longer matches our query (e.g., it was destroyed or a
        // component was removed). Clean up the associated physics body.
        if (store.hasComponent(entity, PhysicsBodyComponent.class)) {
            PhysicsBodyComponent bodyComp = store.getComponent(entity, PhysicsBodyComponent.class);
            physicsEngine.destroyBody(bodyComp.getHandle());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MyRefSystem()`. Systems are managed by the engine's dependency injection or service location framework to ensure they are correctly registered with the ECS scheduler.
- **Unsafe State Modification:** Do not directly call methods on the Store that modify component data (e.g., `store.addComponent(entity, ...)`). This bypasses the engine's deferred command system and will cause unpredictable behavior. Always use the CommandBuffer.
- **Blocking Operations:** The callbacks are executed on a critical path of the game loop. Do not perform file I/O, network requests, or any other long-running, blocking operations within onEntityAdded or onEntityRemove.

## Data Pipeline
The RefSystem acts as a reactive sink in the ECS data flow. It does not produce data but is triggered by changes in the world state.

> Flow:
> World State Change (e.g., `entityManager.addComponent()`) -> ECS Query Processor -> **RefSystem.onEntityAdded** -> CommandBuffer -> Deferred World State Update

