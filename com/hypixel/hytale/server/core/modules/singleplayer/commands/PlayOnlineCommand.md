---
description: Architectural reference for PlayOnlineCommand
---

# PlayOnlineCommand

**Package:** com.hypixel.hytale.server.core.modules.singleplayer.commands
**Type:** Command Object

## Definition
```java
// Signature
public class PlayOnlineCommand extends PlayCommandBase {
```

## Architecture & Concepts
The PlayOnlineCommand class is a concrete implementation of the Command Pattern, designed to handle a specific user-initiated server action. Its sole responsibility is to configure and register the server-side command that transitions a single-player session into an online, publicly accessible game.

This class does not contain any execution logic itself. Instead, it serves as a configuration object for its parent, PlayCommandBase. Upon instantiation, it provides the necessary parameters—the command name "online", a localization key for its description, and a hardcoded access level of **Access.Open**. This configuration instructs the parent class on how to modify the state of the SingleplayerModule when the command is executed by a player.

Architecturally, it is a leaf node in the command system, representing a specific variant of a more generic "play" action. This design decouples the command's definition from its execution, allowing for easy addition of new command variants without altering the core processing logic.

## Lifecycle & Ownership
- **Creation:** An instance of PlayOnlineCommand is created by the SingleplayerModule during its initialization phase. The module is responsible for instantiating and registering all commands relevant to its domain.
- **Scope:** The object's lifecycle is tightly bound to its parent SingleplayerModule. It persists as long as the module is active and the command remains registered in the server's command dispatcher.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the SingleplayerModule is unloaded or the server shuts down, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** The PlayOnlineCommand class is effectively **Immutable**. Its configuration is provided exclusively through its constructor and passed to the superclass. No internal fields can be modified after instantiation.
- **Thread Safety:** This class is inherently **Thread-Safe** due to its immutable nature. The command's execution, which is handled by the parent class, is invoked by the server's main command processing thread. Any state modifications to other systems, such as the SingleplayerModule, must be managed by those systems to ensure thread safety.

## API Surface
The public contract of this class is limited to its constructor, as all functional behavior is inherited from PlayCommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayOnlineCommand(singleplayerModule) | Constructor | O(1) | Constructs a new command instance for the `/play online` action. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is instantiated and managed internally by the SingleplayerModule. A user interacts with this command by typing it into the game console. The following example shows how the system registers it.

```java
// Inside SingleplayerModule's initialization logic
CommandRegistry registry = server.getCommandRegistry();
registry.register(new PlayOnlineCommand(this));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of PlayOnlineCommand using `new`. The command system's integrity depends on it being registered by its owning module. Manual instantiation will result in a non-functional, unregistered command.
- **Direct Execution:** Do not call the inherited `execute` method directly. The server's command processor must be the entry point, as it provides necessary context, permission checks, and argument parsing. Bypassing it can lead to inconsistent server state.

## Data Pipeline
The data flow for this command begins with user input and results in a change to the server's network accessibility state.

> Flow:
> Player Chat Input (`/play online`) -> Server Command Parser -> Command Registry Dispatch -> **PlayOnlineCommand** (Execution delegated to PlayCommandBase) -> SingleplayerModule.setAccess(Access.Open) -> Server Network Listener Update

