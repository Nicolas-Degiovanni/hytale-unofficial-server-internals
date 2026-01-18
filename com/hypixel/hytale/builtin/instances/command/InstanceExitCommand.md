---
description: Architectural reference for InstanceExitCommand
---

# InstanceExitCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Singleton

## Definition
```java
// Signature
public class InstanceExitCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The InstanceExitCommand class implements the server-side logic for the user-facing `/instance exit` command. Architecturally, it serves as a thin, user-facing **Controller** that translates player input into a high-level action within the server's instance management system.

Its primary responsibility is to parse the command context, identify the target player, and delegate the core logic of removing that player from an instance to the **InstancesPlugin**. This design cleanly decouples the command handling and argument parsing layer from the complex state management of game instances, adhering to the Single Responsibility Principle.

The command also demonstrates a common Hytale pattern of using a nested private class, InstanceExitOtherCommand, as a "usage variant". The parent class handles the simple case where a player executes the command on themselves, while the nested class provides the argument parsing and logic for a privileged user (e.g., an administrator) to execute the command on another player.

## Lifecycle & Ownership

-   **Creation:** A single instance of InstanceExitCommand is instantiated by the Command System during server bootstrap or when the parent plugin, InstancesPlugin, is loaded. It is then registered within a central command registry.
-   **Scope:** Application-scoped. The registered command object persists for the entire server session, ready to be executed.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down or the InstancesPlugin is unloaded, at which point it is removed from the command registry.

## Internal State & Concurrency

-   **State:** This class is effectively **stateless and immutable**. Its fields are static final references to pre-configured Message objects for localization. All state required for execution, such as the player and world context, is passed as method parameters during the `execute` call. This design makes the command object inherently reusable and safe.

-   **Thread Safety:** The command object itself is thread-safe. However, its execution methods are subject to strict threading models imposed by the server architecture.
    -   The primary `execute` method, inherited from AbstractPlayerCommand, is guaranteed by the engine to be invoked on the **correct world thread** for the executing player. This provides a safe context for interacting with the player's entity components.
    -   The nested `InstanceExitOtherCommand.executeSync` method demonstrates a critical concurrency pattern. It may be called from a generic command processing thread. To prevent race conditions, it safely retrieves the target player's world and then schedules the core logic for execution on that specific world's thread via `world.execute()`.

    **WARNING:** Any direct manipulation of a player's entity or components must be marshaled to the appropriate world thread. Failure to do so will lead to severe concurrency bugs, data corruption, and server instability.

## API Surface

The public API is not intended for direct developer invocation but is the contract fulfilled for the server's Command System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command for the calling player. Delegates to InstancesPlugin. Throws IllegalArgumentException if the player is not in an instance. |
| executeSync(context) | void | O(1) | *Internal to InstanceExitOtherCommand.* Executes for a targeted player, safely dispatching the logic to the target's world thread. |

*Note: Complexity is O(1) for the command's own logic. The delegated call to InstancesPlugin.exitInstance may have higher complexity.*

## Integration Patterns

### Standard Usage

This class is not designed to be used programmatically. It is registered with the Command System and invoked when a player types the corresponding command in chat.

**Player executing on self:**
> /instance exit

**Administrator executing on another player:**
> /instance exit PlayerName

The Command System receives the chat input, parses it, identifies the registered InstanceExitCommand object, and invokes the appropriate `execute` method with a fully populated CommandContext.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new InstanceExitCommand()`. The command will not be registered with the server and will have no effect. Commands must be managed by the server's Command System.
-   **Manual Invocation:** Do not attempt to call the `execute` method directly. You would be bypassing the server's permission checks, context building, and thread management, which is highly unsafe.
-   **Cross-Thread State Mutation:** The logic within `InstanceExitOtherCommand` explicitly schedules work on the target world's thread. An anti-pattern would be to call `InstancesPlugin.exitInstance` directly from the `executeSync` method without this thread-dispatching step.

## Data Pipeline

The flow of data and control for this command follows a standard server-side command processing pipeline.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parser -> **Command System Registry** -> InstanceExitCommand.execute -> **InstancesPlugin.exitInstance** -> World State Modification -> Entity Manager Update -> Network Packet Sync -> Client World Update

