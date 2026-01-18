---
description: Architectural reference for ClearEntitiesCommand
---

# ClearEntitiesCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class ClearEntitiesCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ClearEntitiesCommand is a server-side command handler that implements the Command Pattern. It is specifically designed to be invoked by a player, as enforced by its base class AbstractPlayerCommand. This class acts as a bridge between the server's command input system and the world state modification logic.

Its primary function is to remove entities from a player-defined volume in the world. Architecturally, it does not contain any logic for defining this volume; instead, it integrates with the **BuilderToolsPlugin** to retrieve the player's current selection state (BlockSelection). This separation of concerns makes the command a pure action handler, delegating state management to the appropriate plugin.

The command operates directly on the core server Entity-Component-System (ECS) by interacting with the World and its underlying EntityStore. This is a high-privilege operation, and access is controlled via the Hytale permission system, as configured in its constructor.

## Lifecycle & Ownership
- **Creation:** A single instance of ClearEntitiesCommand is created and registered by the server's command management system when the **BuilderToolsPlugin** is loaded. It is not created on a per-player or per-execution basis.
- **Scope:** The object instance persists for the entire server session, or until the BuilderToolsPlugin is unloaded. It is a stateless object whose methods are invoked with all necessary context.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the associated plugin is disabled, at which point it is removed from the central command registry.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable** after its initial construction. All data required for an operation, such as the target player and the world state, is passed as arguments to the execute method. The message templates it holds are static and final.
- **Thread Safety:** The instance itself is inherently thread-safe due to its stateless nature. However, the execution logic within the `execute` method is **not thread-safe** and **must** be invoked on the main server thread. It performs direct, unsynchronized modifications to the World and EntityStore, which are not designed for concurrent access. The server's command system guarantees this single-threaded execution context.

## API Surface
The public contract is defined by its role as a command handler. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(V + N) | Executes the entity clearing logic. Complexity is proportional to the volume of the selection (V) plus the number of entities found and removed (N). Throws exceptions on critical failures. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is automatically invoked by the server's command processor when a player with appropriate permissions executes the command in-game.

A conceptual view of the system-level invocation:
```java
// System-level code (conceptual)
// A player types "/clearEntities"
String commandName = "clearEntities";
PlayerRef issuingPlayer = ...;

// The command system finds the registered command object and executes it
Command registeredCommand = commandSystem.find(commandName);
if (registeredCommand instanceof AbstractPlayerCommand) {
    ((AbstractPlayerCommand) registeredCommand).execute(context, store, ref, issuingPlayer, world);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ClearEntitiesCommand()`. The server's command registry manages the lifecycle of this object. Manually creating an instance will result in a non-functional command that is not registered to handle any input.
- **Manual Execution:** Avoid calling the `execute` method directly from other plugin code. This bypasses the command system's permission checks and context setup. If you need to clear entities from a region, use the `World` and `EntityStore` APIs directly.
- **Off-Thread Access:** Never invoke the `execute` method from an asynchronous task or a different thread. Doing so will lead to `ConcurrentModificationException` or, in a worst-case scenario, silent world state corruption.

## Data Pipeline
The flow of data for a typical execution of this command follows a clear path from player input to world modification and back to player feedback.

> Flow:
> Player Input (`/clearEntities`) -> Server Network Layer -> Command Parser -> **ClearEntitiesCommand.execute()** -> BuilderToolsPlugin.getState() -> World.forEachCopyableInSelection() -> EntityStore.removeEntity() -> CommandContext.sendMessage() -> Server Network Layer -> Player Client Chat UI

