---
description: Architectural reference for HitboxCollisionSystems
---

# HitboxCollisionSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.hitboxcollision
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class HitboxCollisionSystems {
    // Contains static inner classes:
    // Setup
    // EntityTrackerUpdate
    // EntityTrackerRemove
}
```

## Architecture & Concepts

The **HitboxCollisionSystems** class is a container for a group of related systems within the Hytale Entity Component System (ECS) framework. It does not represent a single object but rather a logical grouping of systems that collectively manage the lifecycle and network synchronization of the **HitboxCollision** component.

These systems form a cohesive unit responsible for ensuring that hitbox information is correctly initialized on entities, updated, and replicated to all relevant clients. They act as the authoritative bridge between server-side game logic and the client-side representation of entity hitboxes.

The three core systems are:

*   **HitboxCollisionSystems.Setup**: An initialization system that automatically attaches a **HitboxCollision** component to new Player entities based on global server configuration. This ensures players spawn with the correct, default hitbox properties.

*   **HitboxCollisionSystems.EntityTrackerUpdate**: A per-tick network synchronization system. It monitors entities with a **HitboxCollision** component and efficiently sends updates to clients. It sends updates under two conditions: when the hitbox state is marked as dirty (outdated), or when a new client enters an entity's visibility range.

*   **HitboxCollisionSystems.EntityTrackerRemove**: A cleanup and network notification system. It detects when a **HitboxCollision** component is removed from an entity and immediately sends a removal notification to all clients that were tracking that entity. This prevents "ghost" hitboxes on the client.

## Lifecycle & Ownership

The systems within **HitboxCollisionSystems** are designed to be managed by the core ECS engine, not by user code.

*   **Creation:** Instances of **Setup**, **EntityTrackerUpdate**, and **EntityTrackerRemove** are created once during server bootstrap by a central System Registry or Module Loader. They are configured with the necessary **ComponentType** handles at this time.

*   **Scope:** These system objects are singletons that persist for the entire duration of the server session. They are stateless services that operate on data stored within the ECS **Store**.

*   **Destruction:** The systems are discarded and eligible for garbage collection only during server shutdown when the main ECS world is torn down.

## Internal State & Concurrency

*   **State:** The system classes are effectively stateless and immutable after construction. They hold final references to **ComponentType** and **Query** objects, which define their operational scope but do not change during runtime. All mutable state they interact with resides within the ECS components themselves (e.g., the **HitboxCollision** component's internal flags).

*   **Thread Safety:** The systems are designed for concurrent execution by the ECS scheduler.
    *   **EntityTrackerUpdate** explicitly signals its suitability for parallel execution via the **isParallel** method. The ECS framework can safely run its **tick** method on different batches of entities (Archetype Chunks) across multiple threads.
    *   All state mutations (adding or removing components) are deferred through a **CommandBuffer**. This is a critical pattern that queues changes to be applied sequentially after the parallel processing phase, preventing race conditions and ensuring data consistency.

## API Surface

The public API for these systems is defined by the ECS framework interfaces they implement. They are not intended for direct invocation.

### HitboxCollisionSystems.Setup
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Adds a **HitboxCollision** component to a matching entity. |

### HitboxCollisionSystems.EntityTrackerUpdate
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(N) | Framework callback. Executed for each entity matching the query. Queues network updates for viewers. N is the number of viewers. |

### HitboxCollisionSystems.EntityTrackerRemove
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(ref, comp, store, buffer) | void | O(N) | Framework callback. Queues network removal packets for viewers when a **HitboxCollision** component is removed. N is the number of viewers. |

## Integration Patterns

### Standard Usage

Developers do not interact with these systems directly. They are registered with the main ECS **World** or **SystemGroup** during server initialization. The engine's scheduler is then responsible for invoking them at the correct time.

```java
// Example of how these systems would be registered by the engine
// This code is conceptual and resides within the engine's core setup logic.

SystemGroup group = world.getSystemGroup(ServerSystemGroups.ENTITY_TICK);

// The engine instantiates and registers the systems.
group.add(new HitboxCollisionSystems.Setup(...));
group.add(new HitboxCollisionSystems.EntityTrackerUpdate(...));
group.add(new HitboxCollisionSystems.EntityTrackerRemove(...));
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not use `new HitboxCollisionSystems.Setup()` in game logic. The systems are managed entirely by the engine's lifecycle. Manual creation will lead to unregistered systems that never execute.
*   **Manual Network Sync:** Do not attempt to manually replicate **HitboxCollision** state. Rely on the component's internal dirty-flagging (e.g., by calling a method that changes its state) to trigger **EntityTrackerUpdate** automatically. Bypassing this system will cause client-server desynchronization.
*   **Incorrect System Ordering:** The execution order of ECS systems matters. These systems are designed to run in specific groups (e.g., **EntityTrackerSystems.QUEUE_UPDATE_GROUP**). Registering them in the wrong group can break dependencies and lead to unpredictable behavior, such as sending network updates before an entity is actually visible to a client.

## Data Pipeline

The systems orchestrate the flow of hitbox data from server configuration and state changes out to connected clients.

> **Initialization Flow:**
> Player Entity Created -> ECS Query Match -> **HitboxCollisionSystems.Setup** -> CommandBuffer adds **HitboxCollision** component -> Entity is now collidable.

> **Update Flow:**
> Game Logic modifies HitboxCollision state -> `networkOutdated` flag is set to true -> Game Tick -> **HitboxCollisionSystems.EntityTrackerUpdate** -> ComponentUpdate packet created -> Queued for all visible clients -> Network Layer sends packet.

> **Removal Flow:**
> **HitboxCollision** component removed from entity -> ECS Event Triggered -> **HitboxCollisionSystems.EntityTrackerRemove** -> ComponentUpdate (Remove) packet created -> Queued for all previously visible clients -> Network Layer sends packet.

