---
description: Architectural reference for SoundCommand
---

# SoundCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.sound
**Type:** Transient

## Definition
```java
// Signature
public class SoundCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SoundCommand class serves as the root entry point for all sound-related server commands. Architecturally, it functions as a **Command Group** or **Namespace**, delegating specific actions to registered sub-commands rather than containing complex logic itself. Its primary responsibility is to provide the base command, for example `/sound`, and to route subsequent arguments to specialized handlers like SoundPlay2DCommand or SoundPlay3DCommand.

If invoked without any sub-command, its default `execute` method acts as a fallback. This fallback behavior is designed to provide a user-friendly interface, opening the PlaySoundPage UI for the executing player. This pattern decouples the command structure from the user interface, allowing for both script-based and GUI-based interaction with the sound system.

This class is a direct participant in the server's Command System, inheriting from AbstractPlayerCommand to ensure it is correctly registered and that its execution context is populated with the necessary player-specific references.

## Lifecycle & Ownership
- **Creation:** An instance of SoundCommand is created by the server's command registration service during the server bootstrap sequence. The system likely scans for classes annotated or inheriting from a command base and instantiates them once for registration.
- **Scope:** The registered instance persists for the entire server session. It is not created per-execution; a single instance handles all invocations of the `/sound` command.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** SoundCommand is **stateless**. It contains no mutable instance fields that persist between invocations. All necessary state, such as the target player and world context, is provided via the CommandContext argument in the `execute` method. The sub-commands are added during construction and are immutable thereafter.
- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. However, it is designed to be invoked exclusively by the server's main command processing thread. The `execute` method operates on a Store and Ref, which are part of a system that must guarantee safe access to entity components during command execution. Direct invocation from other threads is unsupported and would violate the engine's threading model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SoundCommand() | constructor | O(1) | Initializes the command name and description, and registers its sub-commands. |
| execute(...) | void | O(log N) | Fallback logic. Retrieves player components from the entity store and opens the PlaySoundPage UI for them. Complexity is dominated by component lookups. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by the game engine when a player with sufficient permissions executes the command in the chat console.

```
// Player chat input
/sound
```
The above input triggers the `execute` method, which opens a UI. To trigger a sub-command, the player would provide more arguments.

```
// Player chat input
/sound play2d ...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SoundCommand()` in game logic. The command system handles instantiation and registration. Manually creating an instance will result in a disconnected object that is not part of the command processing pipeline.
- **Manual Execution:** Avoid calling the `execute` method directly. This bypasses the command system's permission checks, context creation, and error handling. To programmatically execute a command, use the server's central command dispatcher service.

## Data Pipeline
The data flow for this command originates from player input and results in a UI state change for that player's client.

> Flow:
> Player Chat Input (`/sound`) -> Network Packet -> Server Command Parser -> **SoundCommand.execute** -> Player.getPageManager() -> Open Custom Page Event -> Network Packet -> Client Renders PlaySoundPage UI

