---
description: Architectural reference for HytalePermissions
---

# HytalePermissions

**Package:** com.hypixel.hytale.server.core.permissions
**Type:** Utility

## Definition
```java
// Signature
public class HytalePermissions {
```

## Architecture & Concepts
The HytalePermissions class serves as a centralized, static repository for all permission strings used throughout the Hytale server. Its primary architectural function is to eliminate the use of "magic strings" when performing security checks, providing a single source of truth for permission nodes.

This class establishes a contract between systems that enforce permissions (such as command handlers or gameplay feature toggles) and the underlying permission management engine that assigns these nodes to players or groups. By defining all permissions as compile-time constants, it ensures type safety, enables IDE auto-completion, and makes refactoring permission structures a safe and manageable process.

The hierarchical naming convention, rooted in the **NAMESPACE** constant, provides a clear and predictable structure for permissions, for example: *hytale.editor.brush.use*.

## Lifecycle & Ownership
- **Creation:** As a utility class composed entirely of static members, HytalePermissions is never instantiated. Its constants and methods are loaded into memory by the JVM ClassLoader when the class is first referenced by another part of the server.
- **Scope:** The class and its members have a static, application-wide scope. They are available for the entire duration of the server's runtime.
- **Destruction:** The class is unloaded from memory when the Java Virtual Machine shuts down. No manual cleanup is required.

## Internal State & Concurrency
- **State:** HytalePermissions is entirely stateless. All fields are declared as **public static final String**, making them immutable compile-time constants. The class holds no mutable data and performs no caching.
- **Thread Safety:** This class is inherently thread-safe. Its immutable, static nature guarantees that it can be safely accessed from any thread without locks or other synchronization primitives. The static factory methods are pure functions, producing predictable output based solely on their inputs.

## API Surface
The public API consists of predefined permission constants and factory methods for constructing dynamic permission nodes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NAMESPACE | String | O(1) | The root namespace for all Hytale permissions: *hytale*. |
| COMMAND_BASE | String | O(1) | The base permission node for all commands: *hytale.command*. |
| *All other constants* | String | O(1) | Predefined permission nodes for specific features like the asset editor or fly camera. |
| fromCommand(name) | String | O(1) | Constructs a standard command permission string. |
| fromCommand(name, sub) | String | O(1) | Constructs a permission string for a command with a subcommand. |

## Integration Patterns

### Standard Usage
This class should be used whenever a system needs to verify if a subject, typically a player, has the authority to perform an action. It is used in conjunction with a permission-checking service.

```java
// Example within a command execution handler
PermissionManager permissionManager = server.getPermissionManager();
Player executingPlayer = ...;

// Check for a static, well-known permission
if (!permissionManager.hasPermission(executingPlayer, HytalePermissions.BUILDER_TOOLS_EDITOR)) {
    executingPlayer.sendMessage("You are not authorized to use the builder tools.");
    return;
}

// Dynamically construct and check a permission for a specific command
String commandName = "gamemode";
if (!permissionManager.hasPermission(executingPlayer, HytalePermissions.fromCommand(commandName))) {
    executingPlayer.sendMessage("You do not have permission to use the /gamemode command.");
    return;
}
```

### Anti-Patterns (Do NOT do this)
- **Hardcoding Permission Strings:** Never use literal strings for permission checks. This creates brittle code that is difficult to maintain and can lead to subtle bugs if the permission hierarchy is ever refactored.
  - **BAD:** `permissionManager.hasPermission(player, "hytale.editor.brush.use")`
  - **GOOD:** `permissionManager.hasPermission(player, HytalePermissions.EDITOR_BRUSH_USE)`
- **Direct Instantiation:** Do not create an instance of this class with `new HytalePermissions()`. It provides no value, as all members are static. Access its members directly via the class name.

## Data Pipeline
HytalePermissions is not a processing node in a data pipeline. Instead, it acts as a static data source, providing constant string values that are consumed by other systems.

> Flow:
> User Input -> Command Dispatcher -> Permission Check Logic -> **HytalePermissions.CONSTANT** -> PermissionManager -> Result (Allow/Deny)

