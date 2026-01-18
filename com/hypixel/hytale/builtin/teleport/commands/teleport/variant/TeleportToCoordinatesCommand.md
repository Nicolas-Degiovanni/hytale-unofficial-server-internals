---
description: Architectural reference for TeleportToCoordinatesCommand
---

# TeleportToCoordinatesCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport.variant
**Type:** Transient

## Definition
```java
// Signature
public class TeleportToCoordinatesCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The TeleportToCoordinatesCommand is a server-side command handler responsible for processing a player's request to teleport to a specific set of world coordinates. It serves as a variant within the broader teleport command group, specifically handling the syntax involving explicit X, Y, and Z coordinates with optional yaw, pitch, and roll for orientation.

Architecturally, this class acts as a translator between player input and the server's Entity Component System (ECS). Its primary responsibility is not to perform the teleport itself, but to validate the input, resolve relative and absolute coordinates, and then signal the *intent* to teleport.

This signaling is achieved by adding a **Teleport** component to the player's entity. This is a critical design pattern that decouples the command parsing logic from the complex, multi-stage process of teleportation. A separate, dedicated TeleportSystem will detect this component later in the server tick, execute the actual position change, handle chunk loading, and perform necessary physics and network synchronization. This ensures that command execution remains lightweight and that teleportation logic is centralized and robust.

## Lifecycle & Ownership

-   **Creation:** A single prototype instance of this command is instantiated by the CommandSystem during server initialization and command registration. It is not created on a per-request basis.
-   **Scope:** The instance persists for the entire server lifetime. However, its execution context, provided via the *execute* method parameters, is ephemeral and lasts only for the duration of a single command invocation.
-   **Destruction:** The prototype instance is discarded during server shutdown when the CommandSystem is torn down.

## Internal State & Concurrency

-   **State:** This class is stateless. Its fields, which define the command's arguments, are final and initialized in the constructor. All state required for execution (e.g., the player, the world, command arguments) is passed into the *execute* method.
-   **Thread Safety:** This class is **not thread-safe**. Like most game-world logic, it is designed to be executed exclusively on the main server thread for a given world. The CommandSystem guarantees this execution context. Any attempt to invoke its methods from a worker thread will lead to race conditions and world state corruption. All interactions with the EntityStore must be synchronized with the main game loop.

## API Surface

The public contract is fulfilled by overriding the *execute* method from its parent, AbstractPlayerCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses coordinates and rotation, then attaches Teleport and TeleportHistory components to the player entity. Complexity is constant for this method, but it triggers downstream systems with high complexity (e.g., chunk loading). |

## Integration Patterns

### Standard Usage

Developers do not instantiate or call this class directly. It is invoked automatically by the server's command processing pipeline when a player with the appropriate permissions executes the corresponding chat command. The system resolves the command, parses arguments, and invokes the *execute* method.

A conceptual view of the system's invocation:

```java
// PSEUDOCODE: How the CommandSystem might dispatch to this command.
// This code does not exist; it is for conceptual understanding only.

Command command = commandRegistry.find("teleport");
CommandContext context = commandParser.parse(playerInput, "/teleport 100 64 -250");

// The system ensures the correct variant is chosen and executed.
if (command instanceof TeleportToCoordinatesCommand) {
    // The system provides the full execution context.
    command.execute(context, world.getStore(), player.getRef(), player.getPlayerRef(), world);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new TeleportToCoordinatesCommand()`. The command must be registered with the server's CommandSystem to function correctly.
-   **Manual Execution:** Do not call the *execute* method directly. This bypasses critical infrastructure, including permission checks, argument parsing, and context validation, and can leave the system in an unstable state.
-   **Direct Position Modification:** The most severe anti-pattern would be to replicate this class's logic but modify the TransformComponent directly. This would break the engine's teleportation pipeline, failing to trigger chunk loading, physics updates, or proper network synchronization, resulting in players falling through the world or being invisible to others.

## Data Pipeline

The flow of data for a teleport command is a multi-stage process that begins with the player and ends with an updated game state, with this class acting as a crucial intermediary.

> Flow:
> Player Command Input -> Network Packet -> Server Command Parser -> **TeleportToCoordinatesCommand** -> Adds *Teleport* Component to Entity -> TeleportSystem (Processes Component) -> Updates *TransformComponent* -> State Synchronization -> Client Render
---

