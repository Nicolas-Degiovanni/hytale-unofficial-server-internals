---
description: Architectural reference for PlayerCameraDemoDeactivateCommand
---

# PlayerCameraDemoDeactivateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCameraDemoDeactivateCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerCameraDemoDeactivateCommand is a concrete implementation within the server's Command System framework. It represents a single, executable action that a privileged user, such as an administrator or the server console, can invoke.

Architecturally, this class follows the **Command Pattern**. It encapsulates a request to deactivate the cinematic camera demo mode as a standalone object. Its primary role is to act as a bridge between the user input layer (the command parser) and the relevant game system, in this case, the CameraDemo singleton.

By extending AbstractTargetPlayerCommand, it leverages a specialized part of the command framework designed for actions that operate on a specific player entity. The parent class handles the complex logic of parsing and resolving the target player from the command arguments, allowing this implementation to focus solely on its core task: invoking the deactivation logic.

## Lifecycle & Ownership
- **Creation:** Instances of this class are not created on-demand. The server's command registration system instantiates it once during the server bootstrap phase. It is then registered within a command tree, likely under a parent command like *camera* and a subcommand *demo*.

- **Scope:** A single instance of PlayerCameraDemoDeactivateCommand persists for the entire lifetime of the server. It is a reusable, stateless object.

- **Destruction:** The object is destroyed and garbage collected only when the server shuts down or if the command is dynamically unregistered from the command system.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. The static field MESSAGE_COMMANDS_CAMERA_DEMO_DISABLED is a constant reference to a translatable message key. All state required for execution (the command source, the target player, the world) is passed into the execute method via the CommandContext.

- **Thread Safety:** This class is inherently **thread-safe**. As it holds no state, multiple threads could theoretically call execute without causing data corruption. However, in practice, the server's command system ensures that commands are processed sequentially on a main logic thread to prevent concurrency issues within the game state itself. The operation's ultimate safety depends on the thread safety of the CameraDemo singleton it invokes.

## API Surface
The public API is defined by the contract of the command system. Direct invocation is discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Overrides the parent method to perform the command's action. Calls CameraDemo.INSTANCE.deactivate() and sends a confirmation message to the command's source. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly by other systems. It is invoked exclusively by the server's command dispatcher in response to a user typing the corresponding command in-game or in the console.

The conceptual flow within the command system is as follows:

```java
// This is a conceptual example. Do not invoke commands this way.
// The Command System handles this process internally.
CommandDispatcher dispatcher = server.getCommandDispatcher();
dispatcher.dispatch("camera demo deactivate SomePlayer", serverConsole);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerCameraDemoDeactivateCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in an object that is not registered and cannot be executed.

- **Manual Execution:** Avoid calling the `execute` method directly. Bypassing the command dispatcher skips critical functionality such as permission checks, argument validation, and context setup, which can lead to NullPointerExceptions or unstable server state.

## Data Pipeline
The data flow for this command begins with user input and ends with a confirmation message and a state change.

> Flow:
> Player Input (`/camera demo deactivate <player>`) -> Network Layer -> Server Command Parser -> Command Dispatcher -> **PlayerCameraDemoDeactivateCommand.execute()** -> CameraDemo Singleton -> Confirmation Message -> Network Layer -> Client Chat UI

