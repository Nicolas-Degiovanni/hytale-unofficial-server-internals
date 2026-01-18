---
description: Architectural reference for PlayerCameraDemoActivateCommand
---

# PlayerCameraDemoActivateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCameraDemoActivateCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerCameraDemoActivateCommand is a concrete implementation within the server's Command System framework. It represents a specific, user-executable action that is dispatched by the central command parser.

As a subclass of AbstractTargetPlayerCommand, this command is designed to operate within the context of a specific player entity. The framework guarantees that when its execution logic is invoked, it will be supplied with valid references to the target player and the world they inhabit.

Architecturally, this class serves as a thin adapter, translating a text-based command from a user into a direct method call on the CameraDemo singleton. It embodies the Command Pattern, where the command itself is an object that encapsulates all information needed to perform an action. This decouples the command invoker (the server's command processing system) from the action's receiver (the CameraDemo system).

## Lifecycle & Ownership
- **Creation:** A single instance of this class is instantiated by the command registration system during server bootstrap. It is registered as a subcommand within the larger camera command structure.
- **Scope:** The command object instance is a long-lived singleton that persists for the entire server session. However, its *execution* is transient and occurs only when a player invokes the corresponding command.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. All required operational data, such as the command sender and target player, is provided as arguments to the execute method. The static MESSAGE field is an immutable constant.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the command system's contract requires that the execute method is only ever called from the main server thread. Any systems it interacts with, such as the CameraDemo singleton or the CommandContext, are not guaranteed to be safe for concurrent access from other threads.

## API Surface
The public API is defined by its parent class, AbstractTargetPlayerCommand. The primary entry point is the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the command logic. Activates the CameraDemo system and sends a confirmation message to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command handling system in response to player input. The framework is solely responsible for instantiating and calling this command with the correct context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerCameraDemoActivateCommand()`. The command system manages the lifecycle of command objects. Manual creation serves no purpose.
- **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context validation, which can lead to server instability or security vulnerabilities.

## Data Pipeline
The execution of this command is the final step in a user-initiated data flow. The flow demonstrates how a player action is translated into a server-side system change.

> Flow:
> Player Chat Input (`/camera demo activate`) -> Network Packet -> Server Command Parser -> Command Dispatcher -> **PlayerCameraDemoActivateCommand.execute()** -> CameraDemo.INSTANCE.activate() -> CommandContext.sendMessage() -> Network Packet -> Client Chat Output

