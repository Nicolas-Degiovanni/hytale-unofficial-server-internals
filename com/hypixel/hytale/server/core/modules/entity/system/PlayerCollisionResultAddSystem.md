---
description: Architectural reference for PlayerCollisionResultAddSystem
---

# PlayerCollisionResultAddSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCollisionResultAddSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerCollisionResultAddSystem is a reactive component within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to ensure that any entity possessing a Player component is also equipped with a default CollisionResultComponent.

This system acts as an **initializer** or a **state guard**. It decouples the logic of player creation from the logic of physics initialization. An entity can be created with only the core Player component, and this system will automatically detect its existence and attach the necessary collision-related data structures.

The core of its mechanism is the `Query` object, which is configured to match entities that have a Player component but do **not** yet have a CollisionResultComponent. When the ECS framework detects a new entity matching this query, it invokes this system's `onEntityAdd` method, thereby completing the entity's setup for physics processing.

This pattern promotes modularity by allowing different systems to be responsible for distinct aspects of an entity's configuration without needing direct knowledge of one another.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level ECS manager during the server's module loading and initialization phase. It is not created on-demand.
-   **Scope:** The instance persists for the entire server session, or as long as the parent entity module is active. It is a long-lived, stateless processor.
-   **Destruction:** De-referenced and garbage collected when the server shuts down or the ECS framework is torn down.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless and immutable** after construction. Its fields are final references to component metadata (ComponentType) and the query definition. It does not store or cache any per-entity or session-specific data.
-   **Thread Safety:** This system is **not thread-safe** and is designed to be operated by a single-threaded ECS update loop. The `onEntityAdd` callback modifies the state of a `Holder` (an entity), an operation that must not be performed concurrently with other reads or writes to the same entity. The parent ECS framework is responsible for ensuring serialized access.

## API Surface
The public API is defined by the HolderSystem contract and is intended for framework invocation, not direct use by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Attaches a new CollisionResultComponent to the entity. |
| onEntityRemoved(holder, reason, store) | void | O(1) | No-op. This system does not react to entity removal. |
| getQuery() | Query | O(1) | Returns the pre-configured query that identifies target entities. |

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this class directly. Instead, it is registered with the server's ECS System Manager during initialization. The framework handles the rest.

```java
// Example: During server module setup
SystemManager systemManager = ...;
ComponentRegistry registry = ...;

// The system is instantiated with dependencies from the registry
PlayerCollisionResultAddSystem system = new PlayerCollisionResultAddSystem(
    registry.get(Player.class),
    registry.get(CollisionResultComponent.class)
);

// The system is registered and will now operate automatically
systemManager.addSystem(system);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call `onEntityAdd` directly. Doing so bypasses the ECS query engine and can lead to inconsistent component state or race conditions if the entity does not actually match the required criteria.
-   **Stateful Implementation:** Do not add mutable fields to this class. Systems in the ECS should be stateless processors that operate on the components passed to them.

## Data Pipeline
This system is triggered by a state change within the `EntityStore`. It does not process a continuous stream of data but rather reacts to discrete events.

> Flow:
> A new entity with a `Player` component is added to the `EntityStore` -> The ECS Framework evaluates its queries against the new entity -> The entity matches this system's query -> **PlayerCollisionResultAddSystem.onEntityAdd()** is invoked -> A new `CollisionResultComponent` is created and attached to the entity's `Holder`.

