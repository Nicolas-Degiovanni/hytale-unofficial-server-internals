---
description: Architectural reference for HideEntitySystems.AdventurePlayerSystem
---

# HideEntitySystems.AdventurePlayerSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** ECS System

## Definition
```java
// Signature
public static class AdventurePlayerSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The AdventurePlayerSystem is a server-side system within Hytale's Entity Component System (ECS) framework. Its sole responsibility is to act as a visibility filter, hiding specific entities from players who are in Adventure mode or have certain client-side settings disabled.

This system does not discover entities. Instead, it operates as a refinement step in a larger data pipeline. It processes a pre-calculated set of potentially visible entities for a given player and prunes this set based on game rules. This design decouples the complex logic of spatial partitioning and visibility calculation (handled by upstream systems like EntityTrackerSystems.CollectVisible) from game-mode-specific presentation rules.

It functions by querying for all entities that are players and then, within its tick logic, iterating through the list of entities visible to that player. If a visible entity is marked with the **HiddenFromAdventurePlayers** component, it is removed from the player's visibility set for the current tick.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS System Scheduler during the bootstrap of the EntityModule. This system is not intended for manual creation by developers.
- **Scope:** The system instance is a singleton for the server's lifetime. Its *tick* method is invoked by the scheduler once per game tick for each entity that matches its component query.
- **Destruction:** The system is destroyed and garbage collected during server shutdown when the ECS world is torn down.

## Internal State & Concurrency
- **State:** This system is fundamentally stateless. Its fields are final references to component types and pre-built query objects. All state modifications occur on the components of the entities it processes, specifically the *visible* collection within the EntityViewer component.
- **Thread Safety:** This system is designed for parallel execution. The `isParallel` method signals to the ECS scheduler that its workload can be distributed across multiple worker threads. Thread safety is achieved because the ECS framework guarantees that each `tick` invocation operates on a unique player entity. Therefore, modifications to one player's EntityViewer component do not conflict with modifications for another.

## API Surface
The public contract of this system is consumed by the ECS scheduler, not by application-level code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, archetypeChunk, store, commandBuffer) | void | O(N) | Executes the filtering logic for a single player entity. N is the number of entities in the player's pre-calculated visibility set. This method is not intended to be called directly. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Integration is achieved by adding the **HiddenFromAdventurePlayers** component to any entity that should be subject to this visibility rule.

```java
// Example: Spawning a "developer-only" marker entity
// This entity will be automatically hidden from players in Adventure mode.
Entity newMarker = commandBuffer.createEntity();
commandBuffer.addComponent(newMarker, new Position(...));
commandBuffer.addComponent(newMarker, new HiddenFromAdventurePlayers());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AdventurePlayerSystem()`. The ECS framework is solely responsible for its lifecycle. Doing so will result in a non-functional object that is not registered with the game loop.
- **Manual Invocation:** Calling the `tick` method manually will bypass the dependency ordering system, query filters, and parallel scheduler. This will almost certainly lead to race conditions, incorrect state, or NullPointerExceptions, as the system expects to run *after* EntityTrackerSystems.CollectVisible has populated the data it relies on.
- **Component Misuse:** Do not add the HiddenFromAdventurePlayers component to critical gameplay entities like other players or hostile mobs, as it will make them invisible to those in Adventure mode, potentially breaking gameplay.

## Data Pipeline
This system is a critical stage in the server's entity visibility and replication pipeline. Its operation is strictly ordered.

> Flow:
> **EntityTrackerSystems.CollectVisible** (Populates a raw list of nearby entities for each player) -> **AdventurePlayerSystem** (Reads the raw list, removes entities with the HiddenFromAdventurePlayers component) -> **Downstream Network Systems** (Read the filtered list and serialize visible entities into network packets for the client)

