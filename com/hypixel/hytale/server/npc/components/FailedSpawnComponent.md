---
description: Architectural reference for FailedSpawnComponent
---

# FailedSpawnComponent

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Component (Tag)

## Definition
```java
// Signature
public class FailedSpawnComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The FailedSpawnComponent is a server-side **Tag Component** within the engine's Entity-Component-System (ECS) architecture. As a tag, it contains no data; its presence on an entity serves as a boolean flag, indicating that the entity's initial spawn attempt was unsuccessful.

This component acts as a critical signaling mechanism between different server systems. A spawner system attaches this component to an NPC entity when it cannot be placed into the world—due to reasons like an invalid location, lack of space, or other gameplay-related constraints. Subsequent systems, such as a cleanup or retry-logic system, can then efficiently query for all entities marked with this component to perform corrective actions.

Its generic type parameter, EntityStore, firmly places it within the server's primary world state management, ensuring it is exclusively used for server-authoritative entities.

### Lifecycle & Ownership
-   **Creation:** Instantiated and attached to an NPC entity by a high-level spawning system at the moment a spawn attempt is determined to have failed. It is never created in isolation.
-   **Scope:** Ephemeral. The component exists on an entity only for the brief period between a failed spawn and the subsequent handling of that failure. It is not intended for long-term persistence.
-   **Destruction:** The component is removed from the entity by a dedicated handler system after the failure has been processed (e.g., after a successful respawn attempt or the entity's permanent deletion). Once detached, the instance is eligible for garbage collection.

## Internal State & Concurrency
-   **State:** **Immutable and Stateless**. This class has no fields. Its entire purpose is defined by its type, not its instance data.
-   **Thread Safety:** **Inherently Thread-Safe**. As a stateless object, the component instance itself poses no concurrency risk. However, all operations that attach or detach this component from an entity **must** be synchronized through the mechanisms provided by the parent EntityStore or world instance to prevent race conditions in the ECS data structures. Direct, unsynchronized modification of an entity's component list is strictly forbidden.

## API Surface
The primary interaction with this component is through its static type accessor, which is used in system queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, engine-registered type identifier for this component. Essential for querying entities. |
| clone() | Component | O(1) | Creates a new instance. This is an internal contract required by the ECS framework for entity duplication. |

## Integration Patterns

### Standard Usage
A system designed to handle spawn failures will query the world's EntityStore for all entities that possess the FailedSpawnComponent.

```java
// Example from a hypothetical NPCRetrySystem
ComponentType<EntityStore, FailedSpawnComponent> failedSpawnType = FailedSpawnComponent.getComponentType();

// Query for all entities that failed to spawn
for (Entity entity : entityStore.getEntitiesWith(failedSpawnType)) {
    // Attempt to respawn, log the failure, or schedule for cleanup
    System.out.println("Processing failed spawn for entity: " + entity.getId());

    // CRITICAL: Remove the component after handling to prevent reprocessing
    entity.removeComponent(failedSpawnType);
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Do not use `new FailedSpawnComponent()` and attempt to manage its state. The component should only be created as part of an `entity.addComponent()` operation.
-   **State Storage:** Do not extend this class to add data. If you need to store information about *why* a spawn failed, create a separate component (e.g., SpawnFailureReasonComponent) with the relevant data fields.
-   **Lingering Components:** Failure to remove this component after handling the failed spawn will result in the entity being processed repeatedly by cleanup systems, causing performance degradation and logical errors.

## Data Pipeline
This component does not process a flow of data. Instead, it represents a state transition within the NPC lifecycle, triggering a specific data-handling path.

> Flow:
> NPC Spawner System -> Spawn Logic Fails -> **FailedSpawnComponent attached to Entity** -> World Tick -> NPC Cleanup System Queries for Component -> Corrective Logic Executed -> **FailedSpawnComponent Removed**

