---
description: Architectural reference for DebugShapeArrowCommand
---

# DebugShapeArrowCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient

## Definition
```java
// Signature
public class DebugShapeArrowCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugShapeArrowCommand is a server-side, player-executable command designed exclusively for debugging purposes. It operates within the server's command processing system and serves as a concrete implementation of the AbstractPlayerCommand, restricting its usage to in-game players.

Its primary function is to provide a visual representation of a player's line of sight. Upon execution, it reads the player's current position, eye height, and head rotation from the Entity Component System (ECS). Using this data, it constructs a transformation matrix and instructs the DebugUtils service to render a temporary arrow shape in the world.

This class is a terminal component in its data flow; it consumes state from the ECS and World systems to produce a transient visual effect, but it does not generate data for other game logic systems. It is a self-contained diagnostic tool.

### Lifecycle & Ownership
- **Creation:** A single instance of DebugShapeArrowCommand is instantiated by the server's command management system during the server bootstrap phase. It is discovered and registered alongside all other command implementations.
- **Scope:** The command object instance is a long-lived singleton that persists for the entire server session, held within the central command registry. The execution context, however, is transient and scoped to a single invocation of the command.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. The only field, MESSAGE_COMMANDS_DEBUG_SHAPE_ARROW_SUCCESS, is a static final constant. All required state for execution (player reference, world, ECS store) is provided as method parameters to the execute method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The execute method is designed to be called from the main server thread, which has exclusive write access to the World and EntityStore. The use of ThreadLocalRandom for color generation is a deliberate choice to avoid contention and locking that would arise from using a shared Random instance, reinforcing its design for safe execution within a multi-threaded server environment.

## API Surface
The public contract is defined by its constructor and the overridden execute method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugShapeArrowCommand() | constructor | O(1) | Instantiates the command with its name and description key. |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the core logic. Reads player transform, calculates orientation, and requests a debug arrow be drawn in the world. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code by developers. It is invoked automatically by the server's command handler when a player with appropriate permissions executes the corresponding command in-game.

**Example Player Action:**
> /debug shape arrow

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DebugShapeArrowCommand()` in game logic. The command system handles the lifecycle of command objects.
- **Manual Invocation:** Calling the `execute` method directly bypasses the command system's permission checks, argument parsing, and context setup. All command execution must be routed through the central command dispatcher.

## Data Pipeline
The command is triggered by player input and results in a visual change in the game world and a feedback message to the player.

> Flow:
> Player Console Input -> Network Packet -> Server Command Parser -> **DebugShapeArrowCommand.execute()** -> DebugUtils.addArrow() -> World State Update -> Network Packet -> Client Renderer

