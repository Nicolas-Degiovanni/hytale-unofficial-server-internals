---
description: Architectural reference for InstanceSpawnCommand
---

# InstanceSpawnCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceSpawnCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InstanceSpawnCommand class is a server-side implementation of the Command Pattern, responsible for handling the player-facing `/instance spawn` console command. It serves as the primary user entry point for the dynamic world instantiation system.

This class acts as a stateless translator, converting raw player input into a structured request for the core **InstancesPlugin**. It is not responsible for the logic of creating worlds or teleporting players; its sole architectural purpose is to parse command arguments, validate them, and delegate the complex, asynchronous task of instance creation to the appropriate service.

It defines three key arguments:
1.  **instanceName:** A required string identifying the instance template to spawn.
2.  **position:** An optional relative coordinate for the spawn location. If omitted, the command executor's current position is used.
3.  **rotation:** An optional rotation vector. If omitted, the command executor's head rotation is used, providing an intuitive default behavior for players.

The command's execution is asynchronous by nature, leveraging a CompletableFuture returned by the InstancesPlugin. This ensures that the main server thread is not blocked by the potentially resource-intensive process of generating and loading a new world.

### Lifecycle & Ownership
-   **Creation:** An instance of InstanceSpawnCommand is created by the server's command registration system during the bootstrap phase, typically when the InstancesPlugin is loaded. It is not created on a per-request basis.
-   **Scope:** The object is a long-lived service handler. It persists for the entire lifecycle of the server session.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the parent plugin is unloaded, which unregisters all its associated commands.

## Internal State & Concurrency
-   **State:** This class is fundamentally **stateless**. The member fields for argument definitions (instanceNameArg, positionArg, rotationArg) are immutable configurations initialized in the constructor. All state required for execution is passed into the `execute` method via the CommandContext and other parameters.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. The `execute` method is designed to be called by the server's main tick loop or a dedicated command-processing thread. It safely dispatches work to other systems, such as the InstancesPlugin, which are responsible for their own concurrency management.

## API Surface
The primary API is not a method to be called directly by developers, but rather the command contract it registers with the server. The framework invokes the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments from the context and initiates an asynchronous instance spawn via the InstancesPlugin. This method returns immediately, offloading the work. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is automatically discovered and used by the server's command processing system. A user triggers its execution by typing the command in chat.

```
// This code is conceptual, representing how the server framework
// would dispatch a parsed command to this handler.

// Player types: /instance spawn my_dungeon ~ ~10 ~
CommandContext context = server.parseCommand(player, "/instance spawn my_dungeon ~ ~10 ~");
InstanceSpawnCommand handler = commandRegistry.getHandlerFor("instance spawn");

// The framework invokes execute
handler.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new InstanceSpawnCommand()`. The command must be registered with the server's command system to function. Direct instantiation creates an orphaned object that will never be used.
-   **Direct Invocation:** Avoid calling the `execute` method directly. Doing so bypasses critical framework features like permission checks, argument parsing, and context setup. Always interact with this functionality through the server's command dispatcher.

## Data Pipeline
The flow of data begins with player input and results in an asynchronous world-generation and player-teleportation sequence.

> Flow:
> Player Chat Input -> Server Command Parser -> **InstanceSpawnCommand** -> InstancesPlugin.spawnInstance -> Asynchronous World Generation -> InstancesPlugin.teleportPlayerToLoadingInstance -> Player Teleportation

