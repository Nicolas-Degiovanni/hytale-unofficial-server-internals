---
description: Architectural reference for WorldEventSystem
---

# WorldEventSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class WorldEventSystem<ECS_TYPE, EventType extends EcsEvent> extends EventSystem<EventType> implements ISystem<ECS_TYPE> {
```

## Architecture & Concepts

The WorldEventSystem is a foundational component within Hytale's Entity-Component-System (ECS) architecture. It serves as a specialized, reactive processor that executes game logic in response to specific world events. Unlike systems that run every frame, a WorldEventSystem is dormant until an event of its subscribed type is dispatched.

Its primary architectural role is to act as the bridge between the event bus and the world state. It provides a structured, safe, and decoupled mechanism for modifying entities and components.

Key architectural concepts include:

*   **Event-Driven Logic:** Systems inheriting from this class do not poll for state changes. They are invoked directly by the engine's event dispatcher when a relevant event, such as PlayerDamageEvent or BlockBrokenEvent, occurs.
*   **State Segregation:** The system enforces a clean separation between reading and writing world state. It receives a read-only **Store** for querying the current state of entities and a **CommandBuffer** for queueing modifications.
*   **Deferred Mutation:** By writing all changes to a CommandBuffer instead of directly to the world state, the engine prevents race conditions and ensures a deterministic order of operations. All queued commands are flushed and applied at a specific synchronization point later in the game tick. This is a critical pattern for stability and performance in a complex ECS.

This class is a generic template, parameterized by an ECS type and an EventType, making it a highly reusable and type-safe foundation for the majority of the game's reactive logic.

### Lifecycle & Ownership

-   **Creation:** Concrete implementations of WorldEventSystem are discovered and instantiated by a central SystemManager or a similar registry during the world-loading bootstrap sequence. They are never created manually by game feature code.
-   **Scope:** An instance of a WorldEventSystem is scoped to a single world instance. It persists for the entire lifetime of that world, from the moment it is loaded until it is fully unloaded (e.g., when a player quits to the main menu).
-   **Destruction:** The system instance is marked for garbage collection when its associated world is unloaded. There are no manual destruction methods to call.

## Internal State & Concurrency

-   **State:** This abstract base class is inherently stateless. Concrete implementations should strive to remain stateless, deriving all necessary information from the incoming event payload and the provided Store.
    **Warning:** Introducing mutable instance fields can lead to severe bugs, especially related to state persisting incorrectly across different events or failing to reset when a world is reloaded.

-   **Thread Safety:** **Not thread-safe.** All methods on a WorldEventSystem instance are designed to be called exclusively from the main world update thread. The ECS framework guarantees this contract.
    **Warning:** Accessing a WorldEventSystem from an asynchronous task or a different thread will lead to data corruption and catastrophic state desynchronization. The CommandBuffer pattern is the designated mechanism for interacting with the world from other threads, not direct system access.

## API Surface

The public contract is focused on implementation by subclasses, not external invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(Store, CommandBuffer, EventType) | abstract void | O(N) | **Implementation Hook.** Subclasses must implement this method to define the core logic for processing the event. Complexity is dependent on the logic. |
| handleInternal(Store, CommandBuffer, EventType) | void | O(N) | **Engine-Facing Entry Point.** Called by the event dispatcher. Contains pre-processing logic before delegating to the abstract handle method. Do not override or call directly. |

## Integration Patterns

### Standard Usage

A developer does not call methods on this class. Instead, they extend it to create a new, specific event handler. The engine's event bus is responsible for routing events to the correct system.

```java
// Example: A system that applies poison damage when a specific event occurs.
public class PoisonDamageSystem extends WorldEventSystem<EcsType, ApplyPoisonEvent> {

    public PoisonDamageSystem() {
        super(ApplyPoisonEvent.class);
    }

    @Override
    public void handle(@Nonnull Store<EcsType> store, @Nonnull CommandBuffer<EcsType> commands, @Nonnull ApplyPoisonEvent event) {
        EntityId targetId = event.getTargetEntity();

        // Read from the world state
        if (store.hasComponent(targetId, HealthComponent.class)) {
            HealthComponent health = store.getComponent(targetId, HealthComponent.class);
            float newHealth = health.getCurrent() - event.getDamageAmount();

            // Queue a change to the world state
            commands.setComponent(targetId, new HealthComponent(newHealth));
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new MyEventSystem()`. The engine's SystemManager is solely responsible for the lifecycle of all systems.
-   **Manual Invocation:** Do not call `handle` or `handleInternal` from other systems or game code. To trigger logic, dispatch an event to the event bus and let the engine route it correctly.
-   **Direct State Mutation:** Never modify the Store object directly. All world state changes **must** be queued through the provided CommandBuffer. Bypassing the buffer will break determinism and cause severe concurrency issues.
-   **Storing Entity References:** Avoid storing entity IDs or component references as instance variables in a system. The entity may be destroyed between event calls, leading to stale references and unpredictable null pointer exceptions.

## Data Pipeline

The WorldEventSystem is a key processing stage in the engine's event data flow. It translates abstract events into concrete changes to the world state.

> Flow:
> Game Action (e.g., Player Input, Network Packet) -> Event Creation (e.g., new ApplyPoisonEvent) -> Event Bus Dispatch -> **WorldEventSystem.handleInternal()** -> CommandBuffer Population -> CommandBuffer Flush -> World State Updated

