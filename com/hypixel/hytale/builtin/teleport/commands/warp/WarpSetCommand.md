---
description: Architectural reference for WarpSetCommand
---

# WarpSetCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Transient

## Definition
```java
// Signature
public class WarpSetCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WarpSetCommand class is a concrete implementation of the Command Pattern, designed to handle the server-side logic for the `/warp set` command. It serves as a direct interface between a player's command input and the underlying warp management system, which is governed by the TeleportPlugin singleton.

Its primary architectural role is to:
1.  **Translate Player Intent:** Convert a chat command into a state-changing operation within the game world.
2.  **Read World State:** Access the Entity Component System (ECS) to retrieve the executing player's current position (TransformComponent) and head rotation (HeadRotation).
3.  **Data Hydration:** Use the retrieved world state to construct a new Warp data object.
4.  **Delegate Persistence:** Pass the newly created Warp object to the TeleportPlugin, which is responsible for managing the in-memory collection of warps and persisting them to the filesystem.

This class is intentionally lightweight and stateless, ensuring that all authoritative warp data remains centralized within the TeleportPlugin.

## Lifecycle & Ownership
-   **Creation:** A single instance of WarpSetCommand is instantiated by the server's command registration system when the TeleportPlugin is loaded. It is not created on a per-request basis.
-   **Scope:** The object instance persists for the entire lifecycle of the server or until the TeleportPlugin is unloaded. The execution context of its methods, however, is scoped to a single command invocation.
-   **Destruction:** The instance is eligible for garbage collection when the command is de-registered, typically during a plugin unload or server shutdown.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its fields, such as nameArg, are final and initialized during construction. It does not cache data or maintain state between executions. All necessary information is provided via the arguments to the execute method.
-   **Thread Safety:** **This class is not thread-safe and must only be invoked from the main server thread.** It performs direct, unsynchronized mutations on the TeleportPlugin's internal warp map and interacts with the non-thread-safe ECS Store. The server's command execution model guarantees single-threaded access, and any deviation from this will lead to state corruption and server instability.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. Complexity is dominated by TeleportPlugin.saveWarps(), which scales with the number of warps (N) being written to disk. Throws exceptions on internal server errors. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command system. The standard interaction is performed by a player in-game.

```
// Player enters the following command in chat
/warp set MyNewHome
```

The server's command dispatcher resolves this input, verifies the player has the *hytale.command.warp.set* permission, parses the "MyNewHome" argument, and invokes the `execute` method on the registered WarpSetCommand instance.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WarpSetCommand()`. The command system manages the lifecycle of command objects. Manual instantiation creates an un-registered command that will never be executed.
-   **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context setup, which can lead to NullPointerExceptions and security vulnerabilities.
-   **Subclassing for Modification:** Do not extend this class to alter its behavior. The command system is not designed for polymorphic dispatch of this nature. To create a custom command, implement a new class from the ground up.

## Data Pipeline
The flow of data for a successful warp creation is linear and synchronous, originating from player input and terminating with a filesystem write and a network message.

> Flow:
> Player Command (`/warp set ...`) -> Server Command Dispatcher -> **WarpSetCommand.execute()** -> Read Player Transform from ECS -> Create `Warp` Object -> Update `TeleportPlugin` In-Memory Map -> `TeleportPlugin.saveWarps()` -> Filesystem Write -> `CommandContext.sendMessage()` -> Network Packet -> Player Client UI

