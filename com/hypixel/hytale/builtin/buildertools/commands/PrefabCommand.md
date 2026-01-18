---
description: Architectural reference for PrefabCommand
---

# PrefabCommand

**Package:** `com.hypixel.hytale.builtin.buildertools.commands`
**Type:** Configuration / Dispatcher

## Definition
```java
// Signature
public class PrefabCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The PrefabCommand class serves as the primary server-side entry point for all prefab-related actions initiated by a player via the command console. It functions as a command dispatcher, aggregating multiple sub-commands (`save`, `load`, `delete`, `list`) into a single `/prefab` namespace. This is an implementation of the Command pattern, where the top-level class does not contain any execution logic itself but instead delegates responsibility to specialized, encapsulated inner classes.

Its core architectural responsibilities are:
1.  **Command Registration:** It defines the base command `prefab` and its alias `p`.
2.  **Permission Scoping:** It establishes a baseline permission requirement, restricting usage to players in Creative mode.
3.  **Sub-Command Aggregation:** It constructs and registers all child commands, such as `PrefabSaveCommand` and `PrefabLoadCommand`, which contain the actual implementation logic.

This class acts as a bridge between the server's command parsing system and the underlying `BuilderToolsPlugin` and `PrefabStore` services. It translates player input into specific, high-level actions within the builder tools ecosystem. The inner command classes handle the complex logic of interacting with the file system, managing different prefab storage locations (asset, server, worldgen), and dispatching tasks to the appropriate world thread.

A key integration point is the command's ability to trigger server-driven UI pages on the client, such as `PrefabPage` and `PrefabSavePage`. This demonstrates that server commands are not limited to headless data processing but can directly control the player's user interface experience.

## Lifecycle & Ownership

-   **Creation:** An instance of PrefabCommand is created once during server initialization, typically as part of the `BuilderToolsPlugin` bootstrap process. It is then registered with the server's central command management system.
-   **Scope:** The object is a long-lived singleton for the duration of the server's runtime. Once registered, it remains active and available to process commands until the server shuts down.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server's command registry is cleared during the shutdown sequence.

## Internal State & Concurrency

-   **State:** The PrefabCommand class itself is stateless and effectively immutable after construction. Its only internal data is the collection of sub-commands, which is populated in the constructor and never modified thereafter. The inner command handler classes are also stateless, processing all required information from the `CommandContext` provided for each execution.
-   **Thread Safety:** The Hytale command system guarantees that the `executeSync` and `execute` methods of all registered commands are invoked on the main server thread. Consequently, this class and its children are not designed for multi-threaded access and contain no explicit synchronization mechanisms. All interactions with world state, entity components, and the `PrefabStore` are performed in a thread-safe manner within the server's primary update loop.

**WARNING:** Calling command execution methods from any thread other than the main server thread will lead to race conditions, world corruption, and server instability.

## API Surface

The public contract of this class is not a programmatic API but the command structure it exposes to players.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| `save` | Sub-Command | O(1) | Opens the prefab saving UI for the player. Requires `hytale.editor.prefab.manage` permission. |
| `load [name] [--children]` | Sub-Command | O(N) | With no arguments, opens the prefab loading UI. With a name, loads the specified prefab. The complexity depends on the size of the prefab being loaded from disk. Requires `hytale.editor.prefab.use` permission. |
| `delete <name>` | Sub-Command | O(1) | Deletes the specified prefab file from the server's prefab directory. Requires `hytale.editor.prefab.manage` permission. |
| `list [storeType] [--text]` | Sub-Command | O(F) | Lists available prefabs. By default, opens a UI. With the `--text` flag, prints a list to chat. Complexity depends on the number of files (F) in the prefab directory. |

## Integration Patterns

### Standard Usage

The class is designed to be used exclusively through the server's command system, invoked by a player with sufficient permissions.

```java
// This is a conceptual example of how the system invokes the command.
// Developers do not write this code; it is handled by the server.

// Player types in chat: "/prefab load my_creation --children"

CommandContext context = createFromPlayerInput("/prefab load my_creation --children");
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.execute(context); // Dispatches to PrefabCommand -> PrefabLoadCommand
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new PrefabCommand()` within game logic. The class is meant to be instantiated once at startup and registered with the server's command framework.
-   **Manual Execution:** Avoid calling the `executeSync` or `execute` methods directly. This bypasses critical infrastructure such as permission checks, argument parsing, and context setup provided by the command system, which can lead to unpredictable behavior and security vulnerabilities.
-   **State Storage:** Do not modify the PrefabCommand instance to store state. It is designed to be a stateless dispatcher. State related to builder tools should be managed in dedicated components like `BuilderToolsPlugin.BuilderState`.

## Data Pipeline

The flow of data for a typical command execution, such as loading a prefab, follows a clear path from player input to world modification.

> Flow:
> Player Chat Input (`/prefab load ...`) -> Server Network Listener -> Command Parsing System -> **PrefabCommand** (Dispatcher) -> `PrefabLoadCommand` (Handler) -> `PrefabStore` (File I/O) -> `BuilderToolsPlugin` (Queue Placement Action) -> World Update -> Client Render

