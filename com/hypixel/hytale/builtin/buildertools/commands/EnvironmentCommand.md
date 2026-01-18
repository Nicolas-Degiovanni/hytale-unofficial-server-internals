---
description: Architectural reference for EnvironmentCommand
---

# EnvironmentCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class EnvironmentCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The EnvironmentCommand class is a server-side implementation of the Command Pattern, designed to handle player-initiated requests to change their perceived world environment. It serves as a direct interface between the player's chat input and the server's environment modification system, specifically for creative mode builder tools.

Architecturally, this class acts as a validator and a delegator. It does not directly manipulate the game state. Its primary responsibilities are:
1.  Defining the command signature, including its name (`environment`), aliases, and required arguments (`environment`).
2.  Enforcing permissions, restricting its use to players in Creative mode.
3.  Parsing and validating the user-provided environment name against the server's asset registry (Environment.getAssetMap).
4.  Providing user-friendly feedback for invalid input, including suggestions for similar, valid environment names.
5.  Delegating the actual environment change operation to the BuilderToolsPlugin's asynchronous update queue.

This delegation is a critical design choice. By queuing the state change, the command decouples itself from the main game loop, ensuring that world modifications are processed in a safe, serialized manner on the appropriate server tick.

## Lifecycle & Ownership
-   **Creation:** An instance of EnvironmentCommand is created by the server's command system during the bootstrap phase of the BuilderToolsPlugin. The system discovers and registers all command handlers provided by the plugin.
-   **Scope:** The registered instance persists for the entire lifecycle of the server or until the owning plugin is disabled. It is a stateless handler that is reused for every execution of the `/environment` command.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the BuilderToolsPlugin is unloaded, which unregisters all its associated commands.

## Internal State & Concurrency
-   **State:** The class contains a single stateful field, `environmentArg`, which holds the definition for the command's required string argument. This field is initialized in the constructor and is treated as immutable for the object's lifetime. The `execute` method itself is stateless, operating exclusively on the arguments provided by the command system.
-   **Thread Safety:** This class is not thread-safe, nor does it need to be. The server's command execution framework guarantees that the `execute` method is always called from the main server thread. The most important concurrency consideration is the handoff to `BuilderToolsPlugin.addToQueue`. This action safely transfers the modification request to a concurrent queue, which is then processed synchronously by the main game loop, preventing race conditions related to world state modification.

## API Surface
The public API is exclusively for consumption by the server's command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Framework entry point. Parses arguments, validates the environment asset, and queues the state change. Throws IllegalArgumentException on internal asset key mismatch. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and managed by the command system. The standard interaction is a player executing the command in the game client.

The framework handles the invocation internally, which would conceptually resemble this:

```java
// PSEUDOCODE: For illustration only. Do not use directly.
CommandSystem commandSystem = server.getCommandSystem();
Command foundCommand = commandSystem.findCommandByName("environment");

// The framework populates the context and dispatches the call
foundCommand.execute(commandContext, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EnvironmentCommand()`. The command will not be registered with the server and will have no effect.
-   **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context setup, which will lead to NullPointerExceptions and unstable server state.

## Data Pipeline
The flow of data and control for a successful command execution follows a clear path from client input to server-side state change.

> Flow:
> Player Chat Input (`/environment overworld`) -> Server Command Parser -> **EnvironmentCommand.execute()** -> Asset Map Lookup -> `BuilderToolsPlugin.addToQueue(lambda)` -> Server Game Tick -> Lambda Execution -> Player State Update -> Network Packet to Client

