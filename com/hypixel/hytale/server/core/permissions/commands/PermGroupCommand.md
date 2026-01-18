---
description: Architectural reference for PermGroupCommand
---

# PermGroupCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands
**Type:** Configuration Object

## Definition
```java
// Signature
public class PermGroupCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PermGroupCommand class serves as a high-level, user-facing administrative interface for the server's core permissions system. It is not a service or a manager, but rather a declarative definition of a command structure that translates text-based user input into direct, state-changing operations on the PermissionsModule.

Architecturally, it follows the **Composite Command** pattern. The primary class, PermGroupCommand, acts as a router or namespace for the `/perm group` command. It contains no execution logic itself. Instead, it aggregates and delegates all functionality to its three private inner subclasses:
*   **PermGroupListCommand:** Handles read operations.
*   **PermGroupAddCommand:** Handles write operations (additions).
*   **PermGroupRemoveCommand:** Handles write operations (removals).

This structure provides a clean separation of concerns, grouping related administrative functions under a single, discoverable entry point while keeping the implementation for each action isolated. The class is a critical bridge between an administrator's intent (expressed via the server console or in-game chat) and the underlying state of the PermissionProvider.

## Lifecycle & Ownership
-   **Creation:** A single instance of PermGroupCommand is created by the server's command registration system during the bootstrap phase. This typically occurs when the parent PermissionsModule is initialized and registers its associated commands with the central CommandManager.
-   **Scope:** The object instance is a long-lived singleton for the duration of the server's runtime. It is held in a central command registry and does not change after initial creation.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down or if the command is explicitly unregistered, which is an uncommon operation.

## Internal State & Concurrency
-   **State:** The PermGroupCommand and its inner command classes are effectively **immutable and stateless** after construction. The constructor establishes the parent-child relationship between the command collection and its subcommands. All execution-specific data, such as the command sender and parsed arguments, is encapsulated within the CommandContext object, which is created fresh for each command execution. The true state is managed entirely by the external PermissionsModule.

-   **Thread Safety:** This component is **not thread-safe** and is designed with a strict single-threaded execution model. The `executeSync` method, as its name explicitly denotes, **must** be invoked on the main server thread. Any attempt to execute command logic from a worker thread will bypass server concurrency controls and lead to race conditions and data corruption within the PermissionsModule. The server's Command Dispatcher is responsible for enforcing this threading constraint.

## API Surface
The primary API is the command-line interface it exposes. The internal `executeSync` methods are the programmatic entry points called by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PermGroupAddCommand.executeSync(context) | void | O(N) | Adds N permissions to a group. Mutates the state of the active PermissionProvider. |
| PermGroupListCommand.executeSync(context) | void | O(P\*M) | Reads and displays all permissions for a group across P providers, each with M permissions. |
| PermGroupRemoveCommand.executeSync(context) | void | O(N) | Removes N permissions from a group. Mutates the state of the active PermissionProvider. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation by other services. Its primary integration point is the server's command registry. The standard usage is by a server administrator via the console or in-game chat.

**Administrator Usage Example:**
```sh
# Add two permissions to the "moderator" group
/perm group add moderator hytale.chat.kick hytale.world.teleport

# List all permissions for the "moderator" group
/perm group list moderator
```

**System Registration Example (Conceptual):**
```java
// During server startup, the CommandManager finds and registers the command.
// This is the ONLY appropriate way to programmatically interact with this class.
CommandManager commandManager = server.getCommandManager();
commandManager.register(new PermGroupCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Never create an instance of this class to call its methods directly. The command system is responsible for its lifecycle and for constructing the required CommandContext for execution.
    ```java
    // ANTI-PATTERN: Bypasses the entire command processing and context pipeline.
    PermGroupCommand cmd = new PermGroupCommand();
    // This is impossible as the inner classes are private and require a valid context.
    ```
-   **Asynchronous Execution:** Do not dispatch command execution to a separate thread. All interactions with the PermissionsModule are assumed to be synchronized with the main server tick.
    ```java
    // ANTI-PATTERN: Will cause severe data corruption.
    new Thread(() -> {
        // Fictional way to get the command and execute it. This will break the server.
        CommandContext fakeContext = createFakeContext();
        command.getSubCommand("add").executeSync(fakeContext);
    }).start();
    ```

## Data Pipeline
PermGroupCommand acts as a control point in a command-and-response flow, not a data processing pipeline.

> **Request Flow:**
> User Input (e.g., `/perm group add ...`) -> Network Layer -> Command Parser -> Command Dispatcher -> **PermGroupCommand** -> PermGroupAddCommand.executeSync -> PermissionsModule.addGroupPermission -> Active PermissionProvider (e.g., Database, File)

> **Response Flow:**
> PermissionsModule -> **PermGroupCommand** -> CommandContext.sendMessage -> Message Formatter -> Network Layer -> User Client (Chat)

