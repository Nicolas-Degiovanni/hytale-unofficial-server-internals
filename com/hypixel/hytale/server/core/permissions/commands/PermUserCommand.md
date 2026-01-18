---
description: Architectural reference for PermUserCommand
---

# PermUserCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands
**Type:** Transient

## Definition
```java
// Signature
public class PermUserCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PermUserCommand class is a command collection that serves as the top-level entry point for all user-permission-related administrative commands. It follows the **Composite Pattern**, acting as a container for more specific command objects (e.g., PermUserAddCommand, PermUserListCommand) and even other collections (PermUserGroupCommand).

Architecturally, this class is a **thin adapter** between the server's Command System and the PermissionsModule. Its sole responsibility is to define the command structure, parse arguments, and delegate the actual business logic to the PermissionsModule singleton. It does not contain any permission logic itself.

The class and its nested subclasses are declarative. They define the command name, description, and expected arguments. The core execution logic is implemented in the `executeSync` method of each final command, which is invoked by the Command System after successful parsing and validation.

## Lifecycle & Ownership
- **Creation:** An instance of PermUserCommand is created once during server bootstrap. The central Command System discovers and instantiates it to register the entire `perm user` command tree.
- **Scope:** The object's lifetime is tied to the server's command registry. It persists for the entire server session.
- **Destruction:** The object is de-referenced and eligible for garbage collection when the server shuts down and the command registry is cleared. It is not created or destroyed on a per-command-execution basis.

## Internal State & Concurrency
- **State:** The PermUserCommand object graph is effectively **immutable** after its constructor completes. It holds no mutable runtime state. Its fields consist of final references to subcommand instances and argument definitions. All stateful operations are forwarded to the PermissionsModule.

- **Thread Safety:** This class is **not thread-safe** and is designed to be accessed only from the main server thread. The `executeSync` method signature is a strong contract indicating that the command executor guarantees synchronous, single-threaded invocation. Any attempts to execute commands from other threads will lead to race conditions and data corruption within the underlying server systems.

## API Surface
The primary API is the command syntax it exposes to a user or console. The programmatic API is the `executeSync` contract for the Command System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| user add | Command | O(N) | Adds N permissions to a user. Delegates to PermissionsModule. |
| user remove | Command | O(N) | Removes N permissions from a user. Delegates to PermissionsModule. |
| user list | Command | O(P) | Lists a user's permissions across P providers. |
| user group add | Command | O(1) | Adds a user to a permission group. |
| user group remove | Command | O(1) | Removes a user from a permission group. |
| user group list | Command | O(P) | Lists a user's groups across P providers. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be discovered and registered by the server's central command handler during startup. A developer would typically add it to a list of commands to be registered.

```java
// In a server bootstrap or module initialization method
CommandRegistry registry = server.getCommandRegistry();
registry.register(new PermUserCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class for any purpose other than registration with the Command System. It provides no useful functionality on its own.
- **Direct Invocation:** Do not call the `executeSync` methods directly. This bypasses the entire Command System's pipeline, including argument parsing, permission checking, and context creation, and will result in a NullPointerException or other undefined behavior.
- **State Storage:** Do not modify this class to store runtime state. It is a definition object, not a service.

## Data Pipeline
The flow of data for a typical command execution is unidirectional, flowing from user input to the core permissioning system.

> Flow:
> User Input (`/perm user add ...`) -> Network Layer -> Command Parser -> **PermUserAddCommand.executeSync** -> PermissionsModule -> Active PermissionProvider -> Data Store (e.g., Config File, Database)

