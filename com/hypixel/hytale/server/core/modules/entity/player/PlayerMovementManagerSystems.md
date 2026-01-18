---
description: Architectural reference for PlayerMovementManagerSystems.AssignmentSystem
---

# PlayerMovementManagerSystems.AssignmentSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System Component

## Definition
```java
// Signature
public static class AssignmentSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The AssignmentSystem is a reactive component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to **bootstrap player entities** by attaching a default MovementManager component.

This system operates as the first stage in a two-part initialization pipeline for player movement. It identifies entities that are designated as players (via the PlayerRef component) but do not yet have movement capabilities. By adding the MovementManager component, it "tags" the entity for further processing by the PostAssignmentSystem in a subsequent phase of the same game tick.

This separation of concerns—attaching a component versus initializing it—is a critical pattern for ensuring deterministic entity setup and avoiding race conditions within the ECS update loop.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the ECS System Registry during server startup. This class is not designed for manual instantiation.
- **Scope:** Singleton per server instance. It persists for the entire lifetime of the server process.
- **Destruction:** The instance is destroyed when the server shuts down and the ECS framework is dismantled.

## Internal State & Concurrency
- **State:** This system is **stateless and immutable**. Its internal fields, QUERY and MOVEMENT_MANAGER_COMPONENT_TYPE, are static final constants. It does not store or cache any per-entity or session-specific data.
- **Thread Safety:** This system is **not thread-safe** and must only be executed by the main server thread that drives the ECS world update. The ECS framework guarantees this execution context, preventing concurrent modification issues.

## API Surface
The public API is intended for consumption by the ECS framework, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static query used to identify player entities lacking a MovementManager. |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Attaches a default MovementManager component to the entity. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Framework callback. This is a no-op; no cleanup is required when the entity is removed. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. It is registered with the server's ECS engine, which automatically calls its methods when an entity's component structure matches the system's query.

```java
// Example of how the framework would register this system (conceptual)
ecsEngine.registerSystem(new PlayerMovementManagerSystems.AssignmentSystem());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AssignmentSystem()`. The ECS engine is responsible for its lifecycle.
- **Manual Invocation:** Never call `onEntityAdd` directly. Doing so bypasses the ECS framework's state management and will lead to an inconsistent world state and unpredictable behavior.

## Data Pipeline
This system acts as a trigger within the entity creation pipeline.

> Flow:
> New Entity Created -> PlayerRef Component Added -> ECS Engine Evaluates Queries -> **AssignmentSystem.onEntityAdd** -> MovementManager Component Attached to Entity

---
---
description: Architectural reference for PlayerMovementManagerSystems.PostAssignmentSystem
---

# PlayerMovementManagerSystems.PostAssignmentSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System Component

## Definition
```java
// Signature
public static class PostAssignmentSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The PostAssignmentSystem is the second and final stage in the player movement initialization pipeline. It is designed to execute immediately after the AssignmentSystem within the same game tick. Its purpose is to perform the **stateful initialization** of the MovementManager component that was previously attached.

This system queries for entities that now possess both a PlayerRef and a MovementManager. Upon finding a match, it retrieves the newly added MovementManager and invokes its `resetDefaultsAndUpdate` method.

A critical architectural element is its use of the CommandBuffer. All component modifications are funneled through the command buffer, which queues the operations. These commands are then flushed and applied deterministically at the end of the tick. This pattern is essential for preventing modification-during-iteration errors and ensuring that all systems in a given tick operate on a consistent snapshot of the world state.

### Lifecycle & Ownership
- **Creation:** Instantiated by the ECS System Registry at server startup. Manual creation is not supported.
- **Scope:** Singleton per server instance, active for the entire server session.
- **Destruction:** Decommissioned during server shutdown as part of the ECS framework's cleanup process.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no mutable fields and all its operations are performed on the components of the entities passed to it by the ECS engine.
- **Thread Safety:** This system is **not thread-safe**. It is designed to be run exclusively on the main server thread. The CommandBuffer it uses is also single-threaded and relies on the ECS engine's synchronized, phased update loop for safe execution.

## API Surface
This API is a contract with the ECS framework and should not be called from other contexts.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static query for player entities that have a MovementManager component. |
| onEntityAdded(ref, reason, store, cmd) | void | O(1) | Framework callback. Initializes the MovementManager component via the CommandBuffer. |
| onEntityRemove(ref, reason, store, cmd) | void | O(1) | Framework callback. This is a no-op. |

## Integration Patterns

### Standard Usage
Like its predecessor, this system is registered with the ECS engine and runs automatically. Its execution is a direct and immediate consequence of the actions performed by the AssignmentSystem.

```java
// Example of how the framework would register this system (conceptual)
ecsEngine.registerSystem(new PlayerMovementManagerSystems.PostAssignmentSystem());
```

### Anti-Patterns (Do NOT do this)
- **Incorrect System Ordering:** Registering this system to run before the AssignmentSystem would render it useless, as its query would never find matching entities during the initial setup tick.
- **Assuming Immediate Initialization:** Code in other systems running in the same tick cannot assume the MovementManager is initialized until after the CommandBuffer has been flushed. This system guarantees initialization by the *end* of the tick, not at the moment of its execution.

## Data Pipeline
This system consumes the output of the AssignmentSystem to complete the data flow for player movement setup.

> Flow:
> AssignmentSystem Attaches Component -> ECS Engine Re-evaluates Queries -> **PostAssignmentSystem.onEntityAdded** -> CommandBuffer.getComponent() -> MovementManager Initialized -> CommandBuffer Queues State Change -> End of Tick: Changes Flushed

