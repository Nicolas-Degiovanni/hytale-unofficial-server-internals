---
description: Architectural reference for PlayerViewRadiusSetCommand
---

# PlayerViewRadiusSetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.viewradius
**Type:** Transient

## Definition
```java
// Signature
public class PlayerViewRadiusSetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerViewRadiusSetCommand is a specific, user-facing command implementation within the server's Command System framework. It acts as a controller, translating a text-based directive from a privileged user (like an administrator) into direct state changes within the server's core Entity Component System (ECS) and the client-facing Network Layer.

Architecturally, this class serves three primary functions:
1.  **Command Definition:** It declares its syntax, including its name (set), required arguments (radius), and optional flags (--blocks, --bypass), to the parent Command System.
2.  **State Mutation:** It directly accesses and modifies components attached to a Player entity. Specifically, it updates the Player component to persist the new view radius setting and the EntityTrackerSystems.EntityViewer component to immediately alter the server's entity tracking logic for that player.
3.  **Client Synchronization:** After mutating server-side state, it constructs and dispatches a ViewRadius network packet. This ensures the player's client is immediately informed of the change, allowing the client-side rendering engine to request or discard chunks accordingly.

This command is a terminal node in the command processing chain, inheriting foundational player-targeting logic from AbstractTargetPlayerCommand.

## Lifecycle & Ownership
-   **Creation:** A single prototype instance of PlayerViewRadiusSetCommand is instantiated by the Command System's registry during server bootstrap. The system discovers and registers all command classes at this time.
-   **Scope:** The command object itself is a long-lived singleton that persists for the entire server session. However, the execution context for any single invocation of the command is extremely short-lived and transient.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the Command System is dismantled.

## Internal State & Concurrency
-   **State:** The command object is effectively stateless between executions. Its internal fields (radiusArg, blocksArg, bypassArg) are final definitions configured in the constructor and are immutable for the lifetime of the object. The command does not cache any data from previous executions. All state it operates on is passed into the execute method via the CommandContext and ECS references.

-   **Thread Safety:** This class is **not thread-safe** and must only be invoked by the main server thread. The Command System guarantees this contract. The execute method performs direct, unsynchronized writes to ECS components (Player, EntityViewer). Calling this method from a worker thread would introduce severe race conditions, leading to data corruption, inconsistent entity tracking, and potential server instability.

## API Surface
The primary public contract is the `execute` method, inherited from its parent and invoked by the Command System. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the view radius modification logic. Throws standard runtime exceptions on assertion failure if ECS components are missing. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by developers. It is designed to be discovered and executed exclusively by the server's Command System in response to in-game or console input.

The conceptual invocation flow is managed entirely by the framework:
```
// PSEUDOCODE: How the Command System invokes this class
CommandSystem.dispatch("player <target> viewradius set <radius> --bypass");

// This eventually resolves to the framework calling:
// commandInstance.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerViewRadiusSetCommand()`. The Command System manages the lifecycle of command objects. Direct instantiation creates an orphan object that is not registered or executable.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical framework functionality, including argument parsing, permission validation, and player targeting logic provided by AbstractTargetPlayerCommand.
-   **Stateful Implementation:** Do not add mutable instance fields to this class to store state between executions. Command objects should be treated as stateless processors.

## Data Pipeline
The flow of data for a successful command execution begins with user input and terminates with a network packet sent to the game client. The PlayerViewRadiusSetCommand is the central processing component in this flow.

> Flow:
> Player Chat Input (`/viewradius set...`) -> Network Ingress -> Command Parser -> **PlayerViewRadiusSetCommand.execute()** -> ECS Store (Update Player & EntityViewer components) -> PlayerRef PacketHandler -> Network Egress (ViewRadius Packet) -> Game Client

