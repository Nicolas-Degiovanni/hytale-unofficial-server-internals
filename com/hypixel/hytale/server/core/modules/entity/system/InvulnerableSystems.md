---
description: Architectural reference for InvulnerableSystems
---

# InvulnerableSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition

```java
// Signature
public class InvulnerableSystems {
    // Nested static classes for ECS Systems and Resources
    public static class EntityTrackerAddAndRemove extends RefChangeSystem<EntityStore, Invulnerable> { ... }
    public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> { ... }
    public static class QueueResource implements Resource<EntityStore> { ... }
}
```

## Architecture & Concepts

The InvulnerableSystems class is a static container for a set of related Entity-Component-System (ECS) constructs that manage the network synchronization of the **Invulnerable** component state for server-side entities. It is a critical part of the server's entity tracking mechanism, ensuring that clients receive timely and efficient updates when an entity's invulnerability status changes.

This system employs a decoupled, two-stage processing model to handle state changes efficiently:

1.  **Detection Stage (EntityTrackerAddAndRemove):** This `RefChangeSystem` acts as an immediate reactor. It listens for the addition or removal of the Invulnerable component on any entity. Upon detection of an *addition*, it does not immediately generate network packets. Instead, it enqueues the entity's reference into a shared, world-scoped **QueueResource**. This decouples the detection of a change from the expensive work of processing and broadcasting it. When a component is *removed*, it directly queues a removal update for all viewers, as this is a simpler, fire-and-forget operation.

2.  **Processing Stage (EntityTrackerUpdate):** This `EntityTickingSystem` runs at a specific point in the server tick, governed by the **EntityTrackerSystems.QUEUE_UPDATE_GROUP**. It processes the buffered entity references from the QueueResource in a batch. For each entity, it generates the appropriate ComponentUpdate packet and dispatches it to all connected clients that are currently tracking that entity. This batching strategy prevents network traffic spikes and ensures that updates are processed predictably within the game loop.

The **QueueResource** serves as the asynchronous communication buffer between the detection and processing stages, enabling robust, concurrent, and performant state synchronization.

### Lifecycle & Ownership

-   **Creation:** The InvulnerableSystems class itself is a static utility and is never instantiated. Its inner system classes, EntityTrackerAddAndRemove and EntityTrackerUpdate, are instantiated by the server's primary ECS framework during the bootstrap process, typically as part of the EntityModule initialization. The QueueResource is also instantiated and registered with the ECS world at this time.
-   **Scope:** Once created, the system and resource instances are registered with the server's main ECS world. They are singletons within that world and persist for the entire duration of the server session.
-   **Destruction:** The systems and resources are destroyed and garbage collected only when the server shuts down and the corresponding ECS world is torn down.

## Internal State & Concurrency

-   **State:** The primary state is managed within the **QueueResource**, which holds a set of entity references (`Set<Ref<EntityStore>>`) that require a network update. This resource is highly mutable, with references being added by one system and consumed by another.

-   **Thread Safety:** This component is designed for a concurrent environment.
    -   The QueueResource internally uses a **ConcurrentHashMap.newKeySet()**, making the queue itself safe for concurrent additions and removals from multiple threads.
    -   The EntityTrackerUpdate system's `isParallel` method can return true, indicating that the ECS scheduler may execute its tick logic across multiple worker threads. The thread-safe nature of the QueueResource is therefore essential to prevent race conditions and ensure data integrity.

    **Warning:** While the queue is thread-safe, logic operating on the components of the entities themselves must adhere to the ECS framework's threading model. Do not assume an entity's components are safe to modify outside of a proper system context.

## API Surface

The public API of this component is not meant for direct invocation but for registration with the ECS framework. The primary interaction point for other systems is retrieving the shared resource.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| QueueResource.getResourceType() | ResourceType | O(1) | Static method to retrieve the unique type identifier for the QueueResource. This is the key used to access the shared queue from the ECS world. |

## Integration Patterns

### Standard Usage

A developer should never interact with these systems directly. The intended pattern is to modify an entity's components, and the systems will react automatically. The framework handles the registration and execution of these systems.

To make an entity invulnerable and have it correctly synchronized to clients, simply add the Invulnerable component to it using a CommandBuffer.

```java
// Correctly trigger the InvulnerableSystems pipeline
// This code would exist within another ECS system.
CommandBuffer<EntityStore> commands = ...;
Ref<EntityStore> entityRef = ...;

// Adding the component is sufficient. EntityTrackerAddAndRemove
// will detect this change and begin the synchronization process.
commands.addComponent(entityRef, new Invulnerable());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new EntityTrackerAddAndRemove()` or `new EntityTrackerUpdate()`. These systems must be instantiated and managed by the ECS world to function correctly.
-   **Manual Queue Manipulation:** Do not access the QueueResource and manually call `queue.add()` or `queue.remove()`. Doing so bypasses the intended logic, breaks the data pipeline, and will lead to network desynchronization or unpredictable behavior.
-   **Assuming Synchronous Updates:** Do not add an Invulnerable component and immediately assume clients have received the update. The update is deferred and processed in a batch during the EntityTrackerUpdate tick. There is a small, intentional latency.

## Data Pipeline

The flow of data for an entity gaining invulnerability is a clear, multi-stage pipeline orchestrated by the ECS framework.

> Flow:
> Game Logic adds **Invulnerable** component -> ECS triggers **EntityTrackerAddAndRemove** -> Entity Ref written to **QueueResource** -> **EntityTrackerUpdate** system tick runs -> Entity Ref read from **QueueResource** -> **ComponentUpdate** packet created -> Packet queued on **EntityViewer** -> Server Network Layer sends packet to Client

