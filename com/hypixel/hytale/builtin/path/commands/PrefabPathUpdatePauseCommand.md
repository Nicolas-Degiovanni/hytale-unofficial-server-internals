---
description: Architectural reference for PrefabPathUpdatePauseCommand
---

# PrefabPathUpdatePauseCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class PrefabPathUpdatePauseCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The PrefabPathUpdatePauseCommand is a server-side command handler responsible for modifying a specific property of a patrol path marker. It serves as a direct interface for server administrators or developers to manipulate entity state in-game without requiring external tools.

Architecturally, this class sits within the server's Command System and acts as a transactional bridge to the Entity Component System (ECS). Its primary function is to parse user-provided arguments, resolve a target entity, and mutate the state of that entity's PatrolPathMarkerEntity component.

Key architectural points include:
- **World-Scoped Operation:** As an AbstractWorldCommand, its execution is always bound to a specific World instance, ensuring all entity operations are performed within the correct simulation context.
- **Flexible Entity Targeting:** The command supports two modes of entity selection: direct specification via an entity ID, or implicit selection via player crosshair targeting (raycasting). This duality is a common pattern in Hytale's command system, handled by utilities like TargetUtil.
- **ECS Interaction:** The command directly interacts with the core ECS data layer through the Store object. It fetches components by type, modifies their internal state, and signals changes to other systems.
- **Change Propagation:** A critical final step is calling `markChunkDirty` on the entity's TransformComponent. This is the integration point with the world persistence and network synchronization systems. It flags the entity's container chunk as modified, ensuring the change to the pause time is saved to disk and replicated to clients.

## Lifecycle & Ownership
- **Creation:** A single instance of PrefabPathUpdatePauseCommand is created by the server's command registration system during server bootstrap or when its containing module is loaded. It is not instantiated per command execution.
- **Scope:** The object instance persists for the entire server session, held in a central command registry.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down or the module is unloaded, at which point it is removed from the command registry.

## Internal State & Concurrency
- **State:** The class instance holds immutable state after construction. The argument definitions, `entityIdArg` and `pauseTimeArg`, are configured in the constructor and are not modified thereafter. The `execute` method is stateless between invocations.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread for the corresponding world. Direct manipulation of the EntityStore and its components from other threads would lead to race conditions and data corruption. The engine's command dispatcher guarantees this single-threaded execution model.

## API Surface
The primary contract is the `execute` method, inherited and implemented from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(log N) | Executes the command logic. Complexity depends on entity targeting; raycasting is typically O(log N) with spatial hashing. Throws GeneralCommandException on argument failure or if the target entity is invalid. |

## Integration Patterns

### Standard Usage
This class is not invoked directly in code. It is triggered by the server's command dispatcher when a user types the corresponding command in the chat console.

```sh
# Set the pause time for the targeted patrol marker to 5.5 seconds
/path update pause 5.5

# Set the pause time for the entity with ID 123 to 10 seconds
/path update pause 10 --entityId 123
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabPathUpdatePauseCommand()`. The command system handles instantiation and registration. A manually created instance would not be registered and would be useless.
- **Manual Execution:** Do not call the `execute` method directly. Bypassing the command dispatcher skips critical steps like permission checking, context setup, and argument pre-processing, which can lead to an unstable server state.

## Data Pipeline
The flow of data for this command begins with user input and ends with a persisted change in the world state.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Dispatcher -> **PrefabPathUpdatePauseCommand.execute()** -> TargetUtil (Raycast) -> EntityStore (Component Lookup) -> PatrolPathMarkerEntity.setPauseTime() -> TransformComponent.markChunkDirty() -> World Persistence & Network Sync Layer

