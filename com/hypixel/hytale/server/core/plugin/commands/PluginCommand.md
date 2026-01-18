---
description: Architectural reference for PluginCommand
---

# PluginCommand

**Package:** com.hypixel.hytale.server.core.plugin.commands
**Type:** Transient

## Definition
```java
// Signature
public class PluginCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PluginCommand class serves as the primary administrative entry point for managing the server's plugin lifecycle via the command system. It is not a monolithic command; rather, it functions as a **Command Collection**, a router that namespaces and delegates execution to a set of specialized, private inner sub-commands. This design pattern keeps the concerns of each action (load, unload, list, etc.) cleanly separated.

Its core architectural responsibilities are:
1.  **Command Namespace:** Establishes the base *plugin* command (and its aliases *plugins*, *pl*) that all plugin-related operations fall under.
2.  **Sub-Command Registration:** In its constructor, it instantiates and registers all its child commands, such as PluginListCommand and PluginLoadCommand. This makes the collection self-contained and easy to integrate into the server's command registry.
3.  **Argument Type Definition:** It defines a static, reusable argument parser, PLUGIN_IDENTIFIER_ARG_TYPE, for converting string input into a structured PluginIdentifier object. This promotes type safety and centralizes the logic for parsing plugin names, ensuring consistency across all sub-commands.

This class acts as a critical bridge between the user-facing command system and the low-level PluginManager service. It translates user intent into direct, state-changing calls on the PluginManager and HytaleServerConfig.

### Lifecycle & Ownership
-   **Creation:** An instance of PluginCommand is created by the server's central command registry during the server bootstrap phase. It is discovered and instantiated as part of the initial command loading process.
-   **Scope:** The object's lifetime is tied to the server session. It persists in the command registry from server start to server shutdown, ready to handle incoming command executions.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The PluginCommand class itself is stateless. Its primary role is to register its children. The inner sub-command classes contain fields like RequiredArg and FlagArg, but these are declarative definitions of the command's structure, not mutable state that persists between executions. The true state that this system manipulates is external, located within the singleton PluginManager and the HytaleServerConfig.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** Command execution is designed to occur synchronously on the server's main thread. The use of `executeSync` in most sub-commands is a strong indicator of this contract. Direct manipulation of the PluginManager or entity components from an asynchronous context is strictly forbidden and will lead to server instability and data corruption.

## API Surface
The primary function of this class is to register its sub-commands. The public contract is fulfilled by its constructor. The effective API is the set of commands it exposes to the server environment.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PluginCommand() | Constructor | O(1) | Instantiates the command collection and registers all sub-commands. |
| list | Sub-Command | O(N) | Lists all currently loaded plugins. Complexity is proportional to the number of plugins. |
| load | Sub-Command | O(1) | Loads a new plugin or adds it to the boot configuration. Involves I/O and can have high latency. |
| unload | Sub-Command | O(1) | Unloads an active plugin or removes it from the boot configuration. Involves I/O. |
| reload | Sub-Command | O(1) | Atomically unloads and then loads a plugin. Involves I/O and can have high latency. |
| manage | Sub-Command | O(1) | Opens a graphical user interface for plugin management for the executing player. |

## Integration Patterns

### Standard Usage
This class is intended to be used via the server console or by an in-game entity with sufficient permissions. It is not designed for programmatic invocation.

```sh
# List all currently loaded plugins
plugin list

# Load a plugin with the identifier "com.example.myplugin"
plugin load com.example.myplugin

# Unload the same plugin and also remove it from the boot list
plugin unload com.example.myplugin --boot

# Reload a plugin to apply new changes
plugin reload com.example.myplugin
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance with `new PluginCommand()`. The server's command registry is solely responsible for its lifecycle. Manually creating an instance will result in a dead object that is not registered to handle any commands.
-   **Direct Execution:** Never call the `executeSync` or `execute` methods on the sub-commands directly. Doing so bypasses the entire command processing pipeline, including argument parsing, permission checks, and context setup.
-   **Stateful Reliance:** Do not assume the state of a plugin remains constant after issuing a command. Operations like `load` and `reload` are complex and may fail. Always check the return messages and verify the plugin's state in the PluginManager if subsequent actions depend on the command's success.

## Data Pipeline
The flow of data for a typical command execution follows a clear path from user input to system state change. The PluginCommand class and its children act as the central dispatch and logic handlers in this flow.

> Flow:
> User Input (`/plugin load myplugin`) -> Server Network Layer -> Command Parsing System -> **PluginCommand** (Routing) -> **PluginLoadCommand** (Execution) -> PluginManager.load(identifier) -> Filesystem I/O -> Plugin Lifecycle Transition -> CommandContext.sendMessage (Feedback to User)

