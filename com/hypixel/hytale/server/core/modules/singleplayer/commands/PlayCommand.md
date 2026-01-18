---
description: Architectural reference for PlayCommand
---

# PlayCommand

**Package:** com.hypixel.hytale.server.core.modules.singleplayer.commands
**Type:** Transient

## Definition
```java
// Signature
public class PlayCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayCommand class is a structural component within the server's command system, implementing the *Composite* design pattern. It does not contain any direct execution logic itself. Instead, its sole responsibility is to act as a top-level namespace and container for a group of related sub-commands: PlayLanCommand, PlayFriendCommand, and PlayOnlineCommand.

This class aggregates functionality related to initiating different single-player session types under a single, user-friendly command, `/play`. When the command system processes a command beginning with "play", it delegates the request to this collection, which in turn routes it to the appropriate sub-command for execution. This design decouples the main command router from the specific implementations of play-related actions, promoting modularity.

The command's dependency on SingleplayerModule is critical; it injects the necessary context into its child commands, allowing them to interact with the single-player game state.

## Lifecycle & Ownership
- **Creation:** An instance of PlayCommand is created by its owning module, SingleplayerModule, during the module's initialization phase. It is not intended to be instantiated by any other system.
- **Scope:** The object's lifetime is strictly bound to the lifetime of the SingleplayerModule instance that creates it. It exists only as long as the single-player game mode is active and its corresponding module is loaded.
- **Destruction:** There is no explicit destruction or cleanup method. The PlayCommand instance is eligible for garbage collection once the SingleplayerModule is unloaded and all references to it from the server's central command registry are cleared.

## Internal State & Concurrency
- **State:** The internal state of a PlayCommand object is effectively immutable after construction. Its primary state is the collection of sub-commands, which is fully populated within the constructor and is not modified thereafter.
- **Thread Safety:** This class is **not thread-safe** and is designed to be accessed exclusively from the main server thread (the game loop tick). The command execution system guarantees single-threaded access, so no internal locking mechanisms are required.

## API Surface
The public contract is minimal, as most functionality is inherited from AbstractCommandCollection and the primary interaction is through the constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayCommand(module) | constructor | O(1) | Constructs the command collection. Requires a non-null SingleplayerModule to pass to its children. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. It is instantiated and registered by its parent module. The example below illustrates the expected creation pattern within the SingleplayerModule.

```java
// Inside SingleplayerModule's initialization logic
CommandSystem commandSystem = context.getService(CommandSystem.class);
PlayCommand playCommand = new PlayCommand(this);
commandSystem.register(playCommand);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of PlayCommand outside of the SingleplayerModule's initialization. Doing so will either fail due to a missing dependency or result in a command that cannot function correctly.
- **State Mutation:** Do not attempt to add or remove sub-commands from this collection after its construction. The command set is considered static for the object's lifetime.

## Data Pipeline
The flow of data for a user-invoked command demonstrates PlayCommand's role as a router.

> Flow:
> User Input (`/play lan`) -> Network Packet -> Server Command Dispatcher -> **PlayCommand** (Router) -> PlayLanCommand (Executor) -> SingleplayerModule (State Change)

