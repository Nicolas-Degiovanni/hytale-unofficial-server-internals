---
description: Architectural reference for TeleportOtherToPlayerCommand
---

# TeleportOtherToPlayerCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport.variant
**Type:** Transient

## Definition
```java
// Signature
public class TeleportOtherToPlayerCommand extends CommandBase {
```

## Architecture & Concepts
The TeleportOtherToPlayerCommand is a server-side command handler that implements a specific variant of the teleport functionality: moving one player entity to another player entity's location. It extends the abstract CommandBase, integrating it directly into the server's command processing system.

This class acts as a high-level orchestrator. It does not contain the low-level logic for entity manipulation. Instead, its primary responsibility is to:
1.  Define and parse the required command arguments (the player to be teleported and the destination player).
2.  Verify the command sender's permissions.
3.  Safely access entity data across potentially different game worlds.
4.  Translate the user's intent into a component-based request for the entity system to process.

The core architectural pattern employed is **Command to Component Transformation**. The command's execution result is not an immediate state change, but the addition of a `Teleport` component to the source player's entity. A separate, dedicated entity processing system will later detect and act upon this component to perform the actual teleportation. This decouples the command input layer from the core game simulation logic.

A critical design element is its handling of cross-world operations. Players can exist in different `World` instances, each with its own dedicated thread and entity store. This command demonstrates the canonical pattern for safe cross-world interaction by scheduling work closures on the appropriate world threads using `World.execute`.

## Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's command registration system during the server bootstrap phase. The system scans for all classes extending CommandBase and instantiates them.
-   **Scope:** The object instance is a long-lived singleton for the duration of the server's runtime. However, its execution context is transient; the `executeSync` method is invoked once per command execution.
-   **Destruction:** The instance is eligible for garbage collection only when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after construction. Its fields are final references to argument definitions (`RequiredArg`) and pre-compiled translation messages (`Message`). All state it operates on is external, provided via the `CommandContext` or retrieved from the game's `World` state.

-   **Thread Safety:** The class instance itself is thread-safe due to its stateless nature. However, the `executeSync` method performs highly sensitive, multi-threaded operations.

    **WARNING:** The logic within `executeSync` is explicitly designed to manage concurrency by "thread hopping". It dispatches lambda functions to the specific threads of the source and target worlds. This is the only safe way to access component data from different worlds. Direct access to an entity's components from a thread other than its owning world's thread will result in data corruption, race conditions, and server instability. The nested `world.execute` calls are a mandatory pattern, not an optimization.

## API Surface
The public API is minimal, intended for use only by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Orchestrates the teleport request. Complexity is proportional to the number of inter-world thread dispatches required (typically 2-3). Sends error or success messages to the command sender. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by developers. It is automatically discovered and executed by the server's command dispatcher in response to a player typing the corresponding command in chat.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportOtherToPlayerCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in a non-functional, unregistered command.
-   **Direct Invocation:** Do not call `executeSync` directly. This bypasses critical infrastructure for argument parsing, permission validation, and context setup provided by the command dispatcher.
-   **Circumventing World Threads:** Replicating the logic of this command without strictly adhering to the `World.execute` pattern for all component access is a critical error. It will break thread-safety guarantees of the entity component system.

## Data Pipeline
This class initiates a command-and-control flow rather than a traditional data processing pipeline.

> Flow:
> Player Command Input -> Command Dispatcher -> **TeleportOtherToPlayerCommand.executeSync()** -> Read Source & Target Player Components (via thread dispatch) -> Add `Teleport` Component to Source Player -> Add `TeleportHistory` Component -> Send Confirmation Message to Sender

The flow is then picked up asynchronously by another system:

> Subsequent Flow:
> Entity Component System (on source world thread) -> Detects `Teleport` Component -> Teleportation Processor -> Updates Entity `TransformComponent` -> Network Sync to Clients

