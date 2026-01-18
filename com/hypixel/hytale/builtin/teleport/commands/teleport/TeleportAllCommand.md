---
description: Architectural reference for TeleportAllCommand
---

# TeleportAllCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient Handler

## Definition
```java
// Signature
public class TeleportAllCommand extends CommandBase {
```

## Architecture & Concepts
The TeleportAllCommand class is a server-side command handler responsible for implementing the logic to teleport all players within a given world to a specific set of coordinates. It serves as a direct bridge between user-initiated chat commands and the server's core Entity Component System (ECS).

Architecturally, this class is not a simple data manipulator. It operates as a state-change *requestor*. Instead of directly modifying an entity's TransformComponent, its primary function is to add a **Teleport** component to every targeted player entity. This is a critical design pattern within the Hytale engine; it decouples the command's intent from the complex mechanics of teleportation (e.g., chunk loading, physics state updates, network synchronization). A separate, dedicated system will later process these Teleport components, ensuring that the operation is performed safely within the main world update loop.

This command also demonstrates the engine's threading model. All entity modifications are marshaled onto the target world's dedicated thread via the `World.execute` method, guaranteeing thread safety and preventing data corruption.

### Lifecycle & Ownership
- **Creation:** An instance of TeleportAllCommand is created by the server's command registration system during the server bootstrap phase or when a game module containing it is loaded. It is not instantiated on a per-command-execution basis.
- **Scope:** The object instance persists for the entire server session, held as a reference within the central command registry.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down or the parent module is unloaded, at which point it is cleared from the command registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its member fields, such as xArg and yArg, are immutable definitions for command argument parsing, configured once in the constructor. All state relevant to a specific execution is contained within the CommandContext object passed to the `executeSync` method.

- **Thread Safety:** The class is thread-safe by design. The entry point, `executeSync`, may be called from a network or command processing thread. However, all state-mutating logic that interacts with the game world is encapsulated within a lambda and submitted to the target World's execution queue. This confines all critical operations to a single, predictable thread, eliminating the risk of race conditions when accessing or modifying entity components.

## API Surface
The public contract is exclusively with the server's command system, which invokes `executeSync` upon a successful command match.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N) | Executes the teleport logic. N is the number of players in the target world. This method is the sole entry point for the command's execution. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke this class directly. Instead, it is registered with the command system, which handles its lifecycle and invocation. The primary interaction is through a player or the server console executing the command.

*Example of user interaction:*
`/tp all 150.5 78.0 -300.0 world_overworld`

The system then performs the following steps:
1.  Parses the input and identifies the TeleportAllCommand handler.
2.  Validates permissions against the `HytalePermissions.fromCommand("teleport.all")` requirement.
3.  Populates a CommandContext with the parsed arguments.
4.  Invokes the `executeSync` method on the registered instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TeleportAllCommand()`. The class is designed to be managed by the command registry. Direct instantiation bypasses argument setup, permission checks, and proper registration, rendering the object non-functional.
- **External Invocation:** Do not call `executeSync` from other parts of the game logic. This violates the command system's design and can lead to unpredictable behavior, especially concerning permissions and sender context. If you need to teleport all players programmatically, create a dedicated service that shares the underlying teleportation logic, rather than invoking the command handler directly.

## Data Pipeline
The flow of data and control for this command begins with user input and ends with entity state modification, mediated by several core systems.

> Flow:
> Player Chat Input -> Network Packet -> Command Parser -> **Command System Dispatcher** -> `TeleportAllCommand.executeSync` -> World Thread Scheduler -> **(On World Thread)** -> Iterate PlayerRefs -> Add `Teleport` component to each player entity -> **(Later, by TeleportSystem)** -> `Teleport` component is processed -> Player's `TransformComponent` is updated -> Network packets are sent to clients.

