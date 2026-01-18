---
description: Architectural reference for IntangibleSystems
---

# IntangibleSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class IntangibleSystems {
    // Note: This class is a non-instantiable container for related ECS systems.
}
```

## Architecture & Concepts

IntangibleSystems is a static container class that groups the core logic for managing an entity's *intangible* state within the server's Entity Component System (ECS) framework. An intangible entity is one that typically bypasses standard physics and interaction, such as a spectator or a ghost.

This class is not a system itself but provides the namespace for two critical systems and a shared resource that work in concert:
1.  **EntityTrackerAddAndRemove:** A reactive system that detects when the Intangible component is added to or removed from an entity.
2.  **EntityTrackerUpdate:** A ticking system that processes intangible state changes and synchronizes them with connected clients via the EntityTrackerSystems.
3.  **QueueResource:** A shared, concurrent data structure used to communicate between the two systems within a single server tick.

The primary architectural role of IntangibleSystems is to bridge the gap between a simple component state change (adding Intangible) and the complex task of network replication. It ensures that all relevant clients (viewers) are correctly notified when an entity they can see becomes intangible, allowing their game client to render or handle it appropriately. This logic is tightly coupled with the EntityTrackerSystems, which manage entity visibility on a per-player basis.

## Lifecycle & Ownership

-   **Creation:** The nested system classes (EntityTrackerAddAndRemove, EntityTrackerUpdate) are not instantiated directly. They are instantiated by the server's ECS framework, likely during the initialization of the EntityModule. The QueueResource is also instantiated and registered with the ECS world's resources.
-   **Scope:** These systems and the associated resource persist for the entire lifetime of the server's primary ECS world (the EntityStore). They are fundamental to the entity tracking and state synchronization engine.
-   **Destruction:** The systems and resource are cleaned up when the ECS world is shut down, typically during a server stop sequence.

## Internal State & Concurrency

-   **State:** The container class IntangibleSystems is stateless. State is managed exclusively within the nested QueueResource class.
    -   **QueueResource:** This resource contains a mutable `Set` of entity references (`Ref<EntityStore>`). This set acts as a temporary, single-frame queue of entities whose intangible state has changed.
-   **Thread Safety:** The systems are designed for concurrent execution.
    -   The QueueResource uses a `ConcurrentHashMap.newKeySet()`, making its queue thread-safe for additions and removals from multiple system execution threads. This is critical because the EntityTrackerUpdate system's `isParallel` method indicates it may be run across multiple threads by the ECS scheduler.
    -   The systems themselves are stateless and operate on data passed to them by the ECS runner (ArchetypeChunks, CommandBuffer), which is a standard pattern for safe parallel execution in an ECS.

## API Surface

The public API consists of the nested classes, which are primarily consumed by the ECS framework itself, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityTrackerAddAndRemove | class | N/A | A RefChangeSystem that reacts to the addition/removal of the Intangible component. |
| EntityTrackerUpdate | class | N/A | An EntityTickingSystem that processes and broadcasts intangible state changes. |
| QueueResource | class | N/A | A world-scoped Resource holding entities that require an intangible state update. |

## Integration Patterns

### Standard Usage

A developer does not interact with IntangibleSystems directly. The standard pattern is to modify an entity's components using a CommandBuffer, which triggers these systems automatically.

```java
// Correctly making an entity intangible
// 'commandBuffer' is provided by the ECS framework
// 'entityRef' is a reference to the target entity

// This action is detected by EntityTrackerAddAndRemove
commandBuffer.addComponent(entityRef, new Intangible());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new IntangibleSystems.EntityTrackerUpdate()`. The ECS framework is responsible for creating, registering, and executing systems. Manual instantiation will result in a non-functional system that is not part of the game loop.
-   **Direct Resource Manipulation:** Do not fetch and modify the QueueResource from general game logic. This resource is an internal implementation detail for communication between the Add/Remove and Update systems within a single tick. Modifying it can lead to desynchronization and unpredictable behavior.
-   **Component-Only Change:** Simply adding the Intangible component is insufficient if the entity is not also tracked by the EntityTrackerSystems (i.e., it lacks the `EntityTrackerSystems.Visible` component). The IntangibleSystems will not process entities that are not being tracked for visibility.

## Data Pipeline

The flow of data for an entity becoming intangible is a multi-stage process orchestrated by the ECS framework.

> Flow:
> 1. Game Logic adds **Intangible** component to an Entity via CommandBuffer.
> 2. ECS framework processes the command, triggering **EntityTrackerAddAndRemove.onComponentAdded**.
> 3. The system adds the entity's reference to the shared **QueueResource**.
> 4. Later in the tick, the ECS scheduler runs the **EntityTrackerUpdate** system.
> 5. **EntityTrackerUpdate.tick** iterates its entities, finds the reference in the QueueResource, and removes it.
> 6. It constructs a **ComponentUpdate** packet of type Intangible.
> 7. It iterates through the entity's `visibleTo` map (from the `EntityTrackerSystems.Visible` component) and queues the update packet for each connected client (EntityViewer).
> 8. The networking layer serializes and sends the packet to clients.
> 9. At the end of its main tick, EntityTrackerUpdate clears the **QueueResource** to prepare for the next frame.

