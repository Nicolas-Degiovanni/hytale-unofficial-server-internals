---
description: Architectural reference for DeployableComponent
---

# DeployableComponent

**Package:** com.hypixel.hytale.builtin.deployables.component
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class DeployableComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DeployableComponent is a data-centric component within Hytale's Entity Component System (ECS) framework. It does not contain behavior logic itself; instead, it holds the state for an entity that represents a "deployable" object, such as a turret, trap, or other player-placed world item.

Its core architectural pattern is the delegation of logic to a configuration object. The component holds a reference to a **DeployableConfig** instance, which implements the actual update logic (the Strategy Pattern). This design separates the *what* (the state of a specific deployable, held in this component) from the *how* (the behavior of a *type* of deployable, held in the config).

This component is designed to be managed exclusively by an associated ECS System (e.g., a DeployableSystem). This system is responsible for iterating over all entities possessing a DeployableComponent and invoking their **tick** method each game loop update.

Key state managed by this component includes:
-   A reference to the owning entity (**owner** and **ownerUUID**).
-   The behavioral configuration (**config**).
-   Lifecycle timestamps (**spawnInstant**).
-   A generic key-value store (**flags**) for managing state machine-like properties without polluting the component with numerous boolean fields.

## Lifecycle & Ownership
-   **Creation:** A DeployableComponent is not instantiated directly. It is added to an entity by a system, typically in response to a player action (e.g., placing a block). The **init** method must be called immediately after creation to populate the component with essential context, such as the deployer's identity and the appropriate DeployableConfig.
-   **Scope:** The lifecycle of a DeployableComponent is strictly bound to the entity it is attached to. It persists as long as the entity exists in the world.
-   **Destruction:** The component is destroyed automatically by the ECS framework when its parent entity is removed from the world. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** This component is highly **mutable**. Its primary purpose is to hold state that changes frequently during gameplay, such as **timeSinceLastAttack** and the values within the **flags** map. It also caches a lazily-initialized **debugColor** for rendering purposes.
-   **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified only by its managing system on the main game thread. Any attempt to read or write its state from an asynchronous task or parallel thread without explicit, external locking will result in race conditions, data corruption, and undefined behavior. The use of ThreadLocalRandom for debug color generation is a strong indicator of its single-threaded access assumption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | **Static.** Retrieves the unique type identifier for this component, used for ECS queries. |
| tick(dt, ...) | void | O(N) | Entry point for the per-frame update logic. Delegates the call to the assigned DeployableConfig. Complexity is dependent on the config's implementation. |
| init(...) | void | O(1) | Initializes the component's state. **CRITICAL:** Must be called once after creation and before the first tick. |
| getOwner() | Ref | O(1) | Returns a reference to the entity that deployed this object. |
| getConfig() | DeployableConfig | O(1) | Returns the behavior configuration object. |
| getFlag(key) | int | O(1) | Retrieves a state flag. Returns 0 if the flag has not been set. |
| setFlag(key, value) | void | O(1) | Sets a state flag to a new integer value. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this component directly. Instead, logic is implemented within a **DeployableConfig** class. The component is then attached to an entity archetype. The managing system handles the lifecycle.

```java
// Example of a system processing this component (conceptual)

// Inside a hypothetical DeployableSystem.onUpdate() method:
Query query = world.createQuery(DeployableComponent.class, Transform.class);

query.forEach((entity, deployable, transform) -> {
    // The system calls tick, not the user.
    // The CommandBuffer is passed in to queue any world changes.
    deployable.tick(dt, entity.getIndex(), ...);
});
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new DeployableComponent()`. Components must be added to entities via the ECS world or a CommandBuffer to be properly registered and managed.
-   **Access Before Initialization:** Reading from this component before its **init** method has been called will result in NullPointerExceptions and invalid state. The component is not in a valid state post-construction.
-   **Manual Tick Invocation:** Do not call the **tick** method from outside the designated managing system. Doing so disrupts the expected order of execution in the game loop and can lead to inconsistent state.
-   **Cross-Thread Modification:** Never modify the component's state from another thread. All changes must be scheduled and executed on the main game thread, typically by queuing commands in a CommandBuffer.

## Data Pipeline
This component acts as a state container within a larger data flow, rather than a pipeline processor itself. Its primary role is to receive time-delta updates and delegate logic that may, in turn, generate commands to alter the game state.

> Flow:
> Game Loop -> DeployableSystem -> **DeployableComponent.tick()** -> DeployableConfig.tick() -> CommandBuffer (Entity modifications, deletions, or creations) -> World State Update

