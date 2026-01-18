---
description: Architectural reference for LocalSpawnBeaconSystem
---

# LocalSpawnBeaconSystem

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** System Component

## Definition
```java
// Signature
public class LocalSpawnBeaconSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnBeaconSystem is a reactive, event-driven component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to monitor the lifecycle of entities that possess the LocalSpawnBeacon component and trigger a global re-evaluation of spawning logic when one is removed.

This system does **not** perform any spawning calculations itself. Instead, it acts as a high-level trigger that decouples the destruction of a spawn point from the complex process of recalculating spawn zones and rates. When an entity with a LocalSpawnBeacon is removed from the world—for example, when a player destroys a physical beacon block—this system intercepts the removal event. It then signals the central LocalSpawnState resource, forcing all associated spawn controllers to immediately re-run their logic.

This design ensures that changes to the world's spawn infrastructure are reflected in the game's behavior without delay, preventing situations where monsters might continue spawning near a recently destroyed beacon.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS framework during server world initialization. Its dependencies, such as the ComponentType and ResourceType handles, are injected by the framework's dependency injection mechanism. This system is not intended for manual creation.
- **Scope:** The lifecycle of a LocalSpawnBeaconSystem instance is bound to a single server world instance (represented by an EntityStore). It persists for the entire duration of the world simulation.
- **Destruction:** The system is decommissioned and marked for garbage collection when its associated world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The LocalSpawnBeaconSystem is **stateless**. Its fields are final references to ECS resource and component types, which are assigned at construction and never change. It does not cache any entity data or maintain state across game ticks.
- **Thread Safety:** This system is **not thread-safe** and is designed to be operated exclusively by the main server thread within the ECS update loop. All method invocations are managed by the framework, which guarantees serialized, single-threaded access to the world state (Store) and CommandBuffer.

## API Surface
The public methods of this class are callbacks intended for invocation by the ECS framework only. Direct calls from game logic are unsupported and will lead to system instability.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Framework callback. Currently a no-op for this system. |
| onEntityRemove(...) | void | O(1) | Framework callback. Triggers a forced re-run of all local spawn controllers via the LocalSpawnState resource. |
| getQuery() | Query | O(1) | Framework callback. Returns a query for all entities with the LocalSpawnBeacon component. |

## Integration Patterns

### Standard Usage
Direct interaction with this system is not possible. Integration is achieved declaratively by adding or removing the LocalSpawnBeacon component from an entity. The system will automatically detect these changes.

```java
// EXAMPLE: Creating an entity that this system will monitor
// This code would exist in a different part of the engine,
// such as world generation or block placement logic.

// When a beacon is placed, a component is added.
// The LocalSpawnBeaconSystem does nothing at this stage.
commandBuffer.addComponent(entityRef, new LocalSpawnBeacon(...));

// Later, when the beacon is destroyed...
// The ECS framework detects the component removal and invokes this system.
commandBuffer.removeComponent(entityRef, LocalSpawnBeacon.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new LocalSpawnBeaconSystem()`. The system requires framework-managed dependencies to function and must be registered with the world's ECS runner to operate.
- **Manual Invocation:** Do not call `onEntityRemove` directly. This bypasses the ECS framework's state management and event queue, which can corrupt the spawning state or cause race conditions. The system's logic relies entirely on being driven by the framework's lifecycle events.

## Data Pipeline
This system acts as an event handler rather than a data processor. Its primary function is to convert a component lifecycle event into a command for another system.

> Flow:
> Entity Destruction -> ECS Framework detects component removal -> **LocalSpawnBeaconSystem::onEntityRemove** -> Accesses LocalSpawnState resource -> Calls `forceTriggerControllers()` -> Spawning logic is re-evaluated globally.

