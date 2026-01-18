---
description: Architectural reference for EntityInteractableSystems
---

# EntityInteractableSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility / System Container

## Definition
```java
// Signature
public class EntityInteractableSystems {
    // Nested System for reacting to component changes
    public static class EntityTrackerAddAndRemove extends RefChangeSystem<EntityStore, Interactable> { ... }

    // Nested System for processing updates each tick
    public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> { ... }

    // Nested Resource for state sharing between systems
    public static class QueueResource implements Resource<EntityStore> { ... }
}
```

## Architecture & Concepts

EntityInteractableSystems is not a traditional object but a container for a group of highly-specialized systems within the server's Entity Component System (ECS) framework. Its sole responsibility is to synchronize the interactable state of an entity between the server and all relevant clients. It ensures that when an entity can be interacted with (e.g., a door can be opened, a vendor can be talked to), the client is notified and can update its UI and input handling accordingly.

This class implements a classic **producer-consumer pattern** to decouple state detection from network packet generation, optimizing performance within a single server tick.

1.  **Producer:** The EntityTrackerAddAndRemove system acts as the producer. It listens for the addition or removal of the Interactable component on entities. When a change is detected, it produces a "work item"—the entity's reference—and places it into a shared buffer.

2.  **Consumer:** The EntityTrackerUpdate system acts as the consumer. It runs later in the server tick, consumes all work items from the shared buffer, and generates the necessary network packets to inform clients of the state change.

3.  **Shared Buffer:** The QueueResource serves as the intermediary, thread-safe buffer between the producer and consumer systems.

This design ensures that network updates are batched and processed at a specific point in the game loop (defined by the QUEUE_UPDATE_GROUP), rather than being sent immediately upon component change. This prevents network flooding and maintains a predictable execution order.

### Lifecycle & Ownership

-   **Creation:** The nested systems (EntityTrackerAddAndRemove, EntityTrackerUpdate) and the QueueResource are instantiated and registered with the server's primary ECS world during the server bootstrap sequence, likely as part of the EntityModule initialization. They are not intended for manual instantiation.
-   **Scope:** These systems and the associated resource are global and persist for the entire lifetime of the server session. Their state is tied to the ECS world itself.
-   **Destruction:** All components are de-referenced and garbage collected when the server shuts down and the ECS world is destroyed.

## Internal State & Concurrency

-   **State:** The primary state is managed within the QueueResource, which holds a transient set of entity references that have changed within the current tick. This state is highly mutable but is intentionally cleared at the end of every tick by the EntityTrackerUpdate system. It is a short-lived communication channel, not a form of persistent storage.

-   **Thread Safety:** **This system is designed for parallel execution.** The QueueResource utilizes a ConcurrentHashMap.newKeySet(), making its queue intrinsically thread-safe for concurrent additions and removals. The EntityTrackerUpdate system explicitly checks if it should run in parallel via the isParallel method. This design allows the ECS scheduler to safely execute these systems across multiple threads to improve server performance, especially with a high number of entities.

    **Warning:** Any modifications to these systems must preserve this thread safety. Direct, unsynchronized access to the underlying queue is strictly forbidden.

## API Surface

The public API is not a set of methods to be called directly, but rather the behavior of the systems as they are managed by the ECS framework. Developers interact with this system *implicitly* by manipulating entity components.

| System / Resource | Type | Role | Description |
| :--- | :--- | :--- | :--- |
| EntityTrackerAddAndRemove | RefChangeSystem | Producer | Reacts to the addition/removal of the Interactable component. Adds entity references to the QueueResource or queues removal packets directly. |
| EntityTrackerUpdate | EntityTickingSystem | Consumer | Runs every tick. Processes entities in the QueueResource and those that are newly visible, generating network updates for clients. |
| QueueResource | Resource | State Buffer | A thread-safe, single-tick communication channel between the AddAndRemove and Update systems. |

## Integration Patterns

### Standard Usage

A developer should never interact with these systems directly. The correct pattern is to modify an entity's components using a CommandBuffer, and the framework will ensure these systems are triggered automatically.

```java
// How a developer should normally use this
// Assume 'entityRef' is a valid reference to an entity
// and 'commandBuffer' is provided by the current system.

// To make an entity interactable:
commandBuffer.addComponent(entityRef, new Interactable());

// The EntityTrackerAddAndRemove system will automatically detect this
// and queue it for a network update via EntityTrackerUpdate.

// To make an entity no longer interactable:
commandBuffer.removeComponent(entityRef, Interactable.getComponentType());

// The system will automatically queue removal packets for relevant clients.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Resource Manipulation:** Do not fetch the QueueResource and modify its contents directly. This breaks the producer-consumer contract and can lead to missed updates or race conditions.
    ```java
    // BAD: Bypasses the system logic
    QueueResource queue = store.getResource(QueueResource.getResourceType());
    queue.queue.add(myEntityRef); // This is incorrect
    ```

-   **Expecting Immediate Updates:** Adding an Interactable component does not send a network packet instantly. The update is deferred until the EntityTrackerUpdate system runs. Logic should not be written with the assumption of immediate client-side state change.

-   **Manual Instantiation:** Never create instances of these systems with `new`. They must be constructed and managed by the ECS framework to be integrated into the server's tick loop and component change notifications.

## Data Pipeline

The flow of data through this system is linear and confined to a single server tick, ensuring that clients are synchronized with the server's state.

> Flow:
> 1. Game Logic adds or removes an **Interactable** component from an entity via a CommandBuffer.
> 2. The ECS framework processes the command, triggering the **EntityTrackerAddAndRemove** system.
> 3. If a component was added, the entity's Ref is added to the **QueueResource**.
> 4. If a component was removed, a "remove" packet is queued directly on the relevant **EntityViewer**.
> 5. Later in the tick, the **EntityTrackerUpdate** system executes.
> 6. The system iterates through the **QueueResource**, creating a **ComponentUpdate** packet for each entity.
> 7. The packet is queued on the **EntityViewer** for each client that can see the entity.
> 8. The server's network layer sends the queued packets to the clients.

