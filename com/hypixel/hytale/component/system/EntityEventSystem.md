---
description: Architectural reference for EntityEventSystem
---

# EntityEventSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract Base Class / System Template

## Definition
```java
// Signature
public abstract class EntityEventSystem<ECS_TYPE, EventType extends EcsEvent> extends EventSystem<EventType> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
The EntityEventSystem is a foundational component of the engine's data-oriented design. It serves as the primary bridge between the global Event Bus and the Entity Component System (ECS). Unlike a standard EventSystem which simply reacts to a broadcast, an EntityEventSystem is a specialized **QuerySystem** that processes events specifically in the context of entities that match a predefined component query.

This class embodies the "Template Method" design pattern. The engine invokes the `handleInternal` method, which performs preliminary event filtering before dispatching to the abstract `handle` method. Subclasses are required to implement the `handle` method, which contains the core business logic for processing a specific event against a single entity within a matched ArchetypeChunk.

Key architectural features include:
*   **Data-Oriented Processing:** By operating on ArchetypeChunks, systems process components that are laid out in contiguous memory blocks, leading to highly efficient, cache-friendly code.
*   **Decoupling:** Systems are completely decoupled from one another. They do not call each other directly, communicating only through events and component data modifications.
*   **Deferred Mutations:** The provided CommandBuffer is the designated mechanism for enacting structural changes to the ECS world (e.g., creating entities, adding components). This prevents iterator invalidation and race conditions during a system's update cycle by queuing operations to be executed at a safe synchronization point.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of EntityEventSystem are not instantiated directly. They are discovered and instantiated by the engine's SystemRegistry during the world initialization or bootstrap phase. This is typically managed via reflection or explicit registration.
- **Scope:** An instance of a system is a singleton scoped to a specific game world (e.g., a server instance or a client session). It persists for the entire lifetime of that world.
- **Destruction:** The system instance is marked for garbage collection when its owning world is unloaded or the engine shuts down. The SystemRegistry manages this process.

## Internal State & Concurrency
- **State:** The EntityEventSystem base class is stateless. However, concrete subclasses are frequently stateful. They may cache configuration data, hold references to other services, or maintain internal state necessary for their logic.
- **Thread Safety:** This class is **not** thread-safe by default. System execution is managed by a scheduler which may run systems in parallel. All interactions with the ECS world state *must* be considered unsafe for concurrent modification.

    **WARNING:** To prevent data corruption and crashes, any operation that modifies the structure of the ECS (adding/removing components, creating/destroying entities) **must** be performed through the CommandBuffer provided to the `handle` method. Direct modification of the Store or ArchetypeChunk is an unsupported and dangerous operation.

## API Surface
The public API is designed for extension by subclasses, not for direct invocation by external callers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(index, chunk, store, buffer, event) | abstract void | O(1) | **Implementer's Entry Point.** Contains the logic to process an event for a single entity. |
| handleInternal(index, chunk, store, buffer, event) | void | O(1) | **Engine's Entry Point.** A non-virtual wrapper that calls the abstract handle method. Do not override or call directly. |

## Integration Patterns

### Standard Usage
Developers do not call an EntityEventSystem directly. Instead, they implement a concrete subclass. The engine's scheduler is responsible for creating the system and invoking its `handle` method for the appropriate entities when a relevant event is fired.

```java
// Example: A system that applies damage to players from a DamageEvent.
// The engine will automatically run this for every entity with a Health component
// when a DamageEvent is dispatched.

public class PlayerDamageSystem extends EntityEventSystem<EcsType, DamageEvent> {

    public PlayerDamageSystem() {
        super(DamageEvent.class);
        // Define the query: this system only runs on entities with a Health component.
        this.setQuery(new QueryBuilder<EcsType>().with(Health.class).build());
    }

    @Override
    public void handle(int entityIndex, ArchetypeChunk<EcsType> chunk, Store<EcsType> store, CommandBuffer<EcsType> commandBuffer, DamageEvent event) {
        // Retrieve the Health component for the current entity
        Health health = store.get(chunk.getEntity(entityIndex), Health.class);

        // Apply the damage
        health.currentValue -= event.getAmount();

        // If health is depleted, queue a command to add a "Dead" tag component.
        if (health.currentValue <= 0) {
            commandBuffer.addComponent(chunk.getEntity(entityIndex), new Dead());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerDamageSystem()`. The SystemRegistry manages the lifecycle. Manually creating an instance will result in a non-functional system that is not registered with the engine.
- **Calling `handle` Manually:** Never invoke the `handle` or `handleInternal` methods directly. This bypasses the engine's scheduler, query system, and synchronization points, leading to unpredictable behavior and state corruption.
- **Direct World Modification:** Avoid modifying component stores or entity lists directly from within the `handle` method. Use the provided CommandBuffer for all structural changes to ensure they are applied safely.

## Data Pipeline
The flow of data for an EntityEventSystem is orchestrated by the engine's main loop.

> Flow:
> External Action (e.g., Player Input) -> Event Dispatcher -> Event Bus -> **EntityEventSystem** (Subscribes to Event) -> ECS Query Engine (Finds matching entities) -> **handle()** method execution for each entity -> CommandBuffer -> Synchronized World Update

