---
description: Architectural reference for PlayFriendCommand
---

# PlayFriendCommand

**Package:** com.hypixel.hytale.server.core.modules.singleplayer.commands
**Type:** Component

## Definition
```java
// Signature
public class PlayFriendCommand extends PlayCommandBase {
```

## Architecture & Concepts
The PlayFriendCommand class is a specialized implementation of the server command pattern, designed to handle a specific user action: connecting to a friend's server from the single-player menu. It extends the generic PlayCommandBase, inheriting the core logic for initiating a server connection.

This class embodies the **Template Method** or **Specialization** pattern. Its primary architectural role is not to introduce new behavior, but to configure its parent, PlayCommandBase, with a specific set of parameters. By passing the command name "friend" and the access level enum Access.Friend to the super constructor, it defines a concrete command variant that the server's command processing system can register and execute.

This design decouples the general connection logic (in PlayCommandBase) from the specific command triggers and permission levels, promoting code reuse and simplifying the addition of new command variations.

### Lifecycle & Ownership
- **Creation:** An instance of PlayFriendCommand is created by the SingleplayerModule during its initialization phase. The module is responsible for instantiating all its associated commands and registering them with a central command handler.
- **Scope:** The object's lifetime is tightly bound to the lifetime of the SingleplayerModule. It persists as long as the single-player context is active and its command is registered.
- **Destruction:** The instance is eligible for garbage collection when the SingleplayerModule is shut down and the server's command registry clears its reference to this command.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no member fields of its own. All state, including the reference to the SingleplayerModule, is managed by the parent PlayCommandBase class. The configuration provided during construction is immutable.
- **Thread Safety:** Command execution in the server is typically marshaled onto a single main logic thread. This class is safe under that assumption. It introduces no new concurrency primitives. Direct, multi-threaded invocation of its methods would be unsafe unless the underlying SingleplayerModule and PlayCommandBase are explicitly designed for it.

## API Surface
The public contract of PlayFriendCommand is defined entirely by its parent, PlayCommandBase. It adds no new public methods. The primary interaction point is the inherited command execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | N/A | *Inherited from PlayCommandBase*. Executes the server connection logic using the parameters defined at construction. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is instantiated and registered by its owning module. The system interacts with it through a generic command interface.

```java
// Example: How the SingleplayerModule might register this command
// This code is conceptual and resides within the module's setup logic.

CommandRegistry registry = server.getCommandRegistry();
SingleplayerModule module = this; // Reference to the owning module

PlayFriendCommand friendCommand = new PlayFriendCommand(module);
registry.register(friendCommand);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually via `new PlayFriendCommand()`. It is designed to be managed by the SingleplayerModule lifecycle. Bypassing the module may lead to a null SingleplayerModule reference and subsequent NullPointerException.
- **Incorrect Context:** Do not attempt to register this command in a context other than a single-player or local server environment. Its dependency on SingleplayerModule makes it fundamentally incompatible with a dedicated server runtime.

## Data Pipeline
The PlayFriendCommand acts as a handler in the server's command processing pipeline. It translates a parsed user input into a call to the single-player connection logic.

> Flow:
> Player Chat Input (`/play friend ...`) -> Server Command Parser -> **PlayFriendCommand** -> PlayCommandBase.execute() -> SingleplayerModule.connectToServer() -> Network Subsystem

