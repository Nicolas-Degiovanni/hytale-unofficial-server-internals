---
description: Architectural reference for NPCSpatialSystem
---

# NPCSpatialSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class NPCSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The NPCSpatialSystem is a specialized component within the server-side Entity-Component-System (ECS) framework. It serves as a bridge between the generic entity data and the engine's spatial partitioning system. Its sole responsibility is to identify all Non-Player Character (NPC) entities and report their world positions to the broader spatial awareness infrastructure.

This system operates on a specific subset of entities defined by its static QUERY field. It targets entities that possess both an NPCEntity component (marking them as an NPC) and a TransformComponent (giving them a position in the world).

By extending the abstract SpatialSystem, this class delegates the complex tasks of managing the spatial data structure (e.g., an octree or spatial hash grid) to its parent. Its primary contribution is implementing the `getPosition` method, which provides the concrete logic for extracting positional data from an NPC's TransformComponent. This design pattern cleanly separates the *what* (NPCs have positions) from the *how* (the underlying spatial grid is managed).

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's ECS World or SystemManager during the world initialization phase. It is not created manually by developers. The required SpatialResource dependency is injected by the engine's service container.
- **Scope:** The instance persists for the entire lifetime of the server world it is registered with. It is a session-scoped object, not a global singleton.
- **Destruction:** The object is marked for garbage collection when the associated server world is unloaded or the server shuts down. The ECS framework manages its lifecycle completely.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It does not maintain any internal caches or mutable fields related to game state. All data is read directly from the ArchetypeChunks passed into its methods during the system update tick. The true state is owned by the ECS Store and the parent SpatialSystem.
- **Thread Safety:** This system is **not thread-safe** and is designed to be executed by a single-threaded, deterministic game loop scheduler. The `tick` method must not be invoked concurrently from multiple threads. The underlying ECS framework may process entity chunks in parallel, but the system's update logic itself is expected to be synchronous within its designated update phase.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| QUERY | Query | O(1) | A static definition of the entity components this system will process. |
| getQuery() | Query | O(1) | Returns the static QUERY object. Used by the ECS scheduler to identify relevant entities. |
| tick(dt, systemIndex, store) | void | O(N) | The main update entry point called by the engine each frame. N is the number of entities matching the QUERY. It delegates processing to the parent SpatialSystem. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Extracts the world position from the TransformComponent of a specific entity within a data chunk. This is the core logic of the class. |

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this class directly. Instead, it is registered with the server's system manager during world setup. The engine handles its entire lifecycle and execution.

```java
// Example: Registering the system during server initialization
// (This is a conceptual example of engine-level code)

SystemManager manager = world.getSystemManager();
SpatialResource spatialResource = world.getResource(SpatialResource.class);

// The engine creates and registers the system.
// Developers do not call 'new' themselves.
manager.register(new NPCSpatialSystem(spatialResource));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new NPCSpatialSystem()`. The `SpatialResource` dependency is managed by the engine and must be injected correctly. Manual creation will lead to a non-functional system that is disconnected from the game world.
- **Manual Ticking:** Never call the `tick` method directly. The engine's scheduler invokes systems in a precise order to ensure data consistency. Manual invocation will bypass this and cause unpredictable behavior, such as operating on stale positional data or creating race conditions with other systems.

## Data Pipeline
This system acts as a data provider, feeding positional information from the ECS into the abstract spatial partitioning system.

> Flow:
> ECS Store -> **NPCSpatialSystem** (filters for NPCs) -> `getPosition` (extracts Vector3d) -> `SpatialSystem` (parent class) -> `SpatialResource` (updates the underlying octree/grid)

