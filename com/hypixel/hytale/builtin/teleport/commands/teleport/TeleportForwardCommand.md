---
description: Architectural reference for TeleportForwardCommand
---

# TeleportForwardCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class TeleportForwardCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TeleportForwardCommand class is a concrete implementation within the Server Command System, encapsulating the logic for the user-facing `/forward` command. It serves as a direct bridge between player chat input and the Entity Component System (ECS).

Its primary architectural role is to translate a parsed command into a state change on a specific component, in this case, the TeleportHistory. By extending AbstractPlayerCommand, it integrates seamlessly into the server's command discovery and dispatching mechanism, inheriting boilerplate logic for player-context validation and permission enforcement. This class is a classic example of the Command Pattern, isolating the request for an action (teleporting forward) into a standalone object.

## Lifecycle & Ownership
- **Creation:** A single instance of TeleportForwardCommand is instantiated by the server's command registration system during the server bootstrap phase. It is discovered via classpath scanning and registered with a central command dispatcher.
- **Scope:** The object is a singleton that persists for the entire server session. It does not hold per-player state; its instance is shared across all command executions.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down or the command is explicitly unregistered from the dispatcher.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, countArg, is an immutable definition of a command argument configured during construction. All stateful operations, such as modifying teleport history, are performed on components passed into the execute method. The state is owned by the player entity, not the command object.
- **Thread Safety:** The command instance itself is thread-safe due to its stateless nature. The execute method is designed to be called by the server's main thread or a designated command processing thread. Concurrency guarantees for the underlying entity and component data are managed by the ECS Store, not this class. Direct, concurrent invocation of the execute method for the same player is not a supported or safe operation.

## API Surface
The public contract is almost entirely defined by the inherited AbstractPlayerCommand. The primary entry point is the execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command. Retrieves the TeleportHistory component from the player entity and invokes its forward method. Throws exceptions if the component is missing or if arguments are malformed. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command system. A user triggers this logic by typing the command into the game client.

> **Player Input:** `/forward 2`

The system handles parsing, permission checks, and invocation.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TeleportForwardCommand()`. An instance created this way will not be registered with the command dispatcher and will have no effect.
- **Manual Invocation:** Calling the `execute` method directly bypasses critical infrastructure, including argument parsing, permission validation, and context setup. This will lead to unpredictable behavior, likely resulting in a NullPointerException or an invalid game state.

## Data Pipeline
The flow of data for a typical `/forward` command execution demonstrates how this class fits into the broader server architecture.

> Flow:
> Player Chat Packet -> Network Layer -> Command Dispatcher -> Permission Check -> **TeleportForwardCommand.execute()** -> ECS Store (GetComponent) -> TeleportHistory.forward() -> ECS Store (SetComponent) -> World State Update -> Network Sync to Clients

