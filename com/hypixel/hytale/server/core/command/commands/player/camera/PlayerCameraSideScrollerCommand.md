---
description: Architectural reference for PlayerCameraSideScrollerCommand
---

# PlayerCameraSideScrollerCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCameraSideScrollerCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerCameraSideScrollerCommand is a server-side command handler responsible for enforcing a specific 2.5D "side-scroller" camera perspective on a target player's client. It is a concrete implementation within the server's command processing system and extends AbstractTargetPlayerCommand, which provides the necessary boilerplate for parsing a target player from the command's arguments.

Its core function is to construct a highly specific ServerCameraSettings configuration object and transmit it to the client using a SetServerCamera packet. This configuration locks the player's view to a fixed plane, restricts movement along the Z-axis, and alters mouse input to interact with that plane. This class serves as a powerful tool for game designers to create specific gameplay sequences, cutscenes, or mini-games without requiring custom client-side code. It is a prime example of the server's authority to dictate client rendering state.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the Command System during server bootstrap. It is then registered with the command dispatcher under the name "sidescroller".
- **Scope:** The command object is a long-lived singleton that persists for the entire server session. However, each execution of the command is an ephemeral, stateless operation.
- **Destruction:** The instance is destroyed when the server shuts down and the Command System de-registers all commands.

## Internal State & Concurrency
- **State:** This class is stateless. The `execute` method operates exclusively on its parameters and local variables. The ServerCameraSettings object is created anew for every invocation, ensuring no state is carried between command executions.
- **Thread Safety:** The class is inherently thread-safe. As a stateless handler, concurrent executions for different players will not interfere with one another. However, the parameters passed to the `execute` method, such as the World and PlayerRef objects, are managed by the core game loop and must be accessed according to the engine's threading model. Direct modification of these objects from an asynchronous context is not supported.

## API Surface
The public contract is defined by the parent AbstractTargetPlayerCommand class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, source, target, player, world, store) | void | O(1) | Constructs and sends a SetServerCamera packet to the target player. This is the primary entry point, called by the command system. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from code. It is designed to be triggered by a privileged user (e.g., an administrator) or a server-side script via the command console. The command system handles parsing, target resolution, and invocation.

*User-facing invocation:*
```sh
/camera sidescroller <playerName>
```

*System-level invocation (conceptual):*
```java
// The Command System finds the registered instance and invokes it.
// Developers should NOT do this manually.
Command command = commandRegistry.get("sidescroller");
command.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerCameraSideScrollerCommand()`. Doing so bypasses the command registration and dispatching system, including critical permission checks and argument parsing handled by the framework.
- **Manual Execution:** Do not call the `execute` method directly. This subverts the command context and player targeting logic provided by the AbstractTargetPlayerCommand superclass, leading to unpredictable behavior and potential NullPointerExceptions.

## Data Pipeline
The command initiates a one-way flow of data from the server to a specific client, altering the client's rendering behavior.

> Flow:
> Server Command Input -> Command Parser -> **PlayerCameraSideScrollerCommand** -> Packet Handler -> SetServerCamera Packet -> Network Layer -> Client Camera System -> Client Renderer Update

