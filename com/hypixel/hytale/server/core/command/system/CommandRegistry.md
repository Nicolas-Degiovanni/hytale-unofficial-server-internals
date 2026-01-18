---
description: Architectural reference for CommandRegistry
---

# CommandRegistry

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class CommandRegistry extends Registry<CommandRegistration> {
```

## Architecture & Concepts
The CommandRegistry is a specialized, scoped container responsible for managing the lifecycle of server commands associated with a specific plugin. It serves as a critical bridge between an individual plugin's command definitions and the global, server-wide CommandManager.

Its primary architectural functions are:
1.  **Ownership Association:** It automatically stamps each registered command with a reference to its owning plugin. This is essential for permission handling, lifecycle management, and server administration.
2.  **Precondition Enforcement:** By extending the base Registry class, it inherits a precondition mechanism. This ensures that plugins can only register commands during designated, safe phases of the server lifecycle, preventing state corruption.
3.  **Lifecycle Scoping:** It groups all command registrations from a single plugin. When a plugin is disabled or reloaded, the server can use this registry to cleanly unregister all associated commands without affecting other plugins.

This class is not a global singleton; rather, a distinct instance is created for each plugin that needs to register commands, providing a clean separation of concerns.

### Lifecycle & Ownership
-   **Creation:** An instance of CommandRegistry is created by the server's plugin loading system. It is then provided to a plugin, typically during its enablement phase. It is not intended to be instantiated directly by plugin developers.
-   **Scope:** The object's lifetime is tightly coupled to the lifetime of the plugin it serves. It persists as long as the plugin is enabled.
-   **Destruction:** The CommandRegistry is marked for garbage collection when the associated plugin is disabled. The server framework is responsible for iterating through its contents to unregister all commands from the global CommandManager prior to destruction.

## Internal State & Concurrency
-   **State:** The CommandRegistry is stateful. It maintains a reference to its owning Plugin and inherits a collection of CommandRegistration objects from its parent. This state is mutable, growing as `registerCommand` is called.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. Command registration is a sensitive operation that is expected to occur exclusively on the main server thread during a plugin's initialization sequence. Accessing it from other threads will lead to race conditions and unpredictable behavior in the global CommandManager.

## API Surface
The public API is minimal, exposing only the functionality necessary for a plugin to register its commands.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerCommand(AbstractCommand command) | CommandRegistration | O(1) | Registers a command with the global system and associates it with the owning plugin. Throws an IllegalStateException if the registration precondition fails. |

## Integration Patterns

### Standard Usage
The intended use is for a plugin to receive an instance of CommandRegistry from the server framework and use it to register all its commands during its startup logic.

```java
// Example from within a Plugin's onEnable method
public void onEnable(PluginContext context) {
    CommandRegistry commandRegistry = context.getCommandRegistry();

    commandRegistry.registerCommand(new HealCommand());
    commandRegistry.registerCommand(new GamemodeCommand());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new CommandRegistry()`. The framework must provide a correctly configured instance with the proper plugin context and lifecycle preconditions. Direct instantiation will result in commands that are not correctly owned or managed.
-   **Late Registration:** Do not hold a reference to the CommandRegistry and attempt to register commands after the plugin's enablement phase is complete. All registrations should occur synchronously during initialization to ensure a consistent and predictable server state.
-   **Cross-Plugin Registration:** Do not pass a CommandRegistry instance from one plugin to another. Each registry is scoped to a single plugin, and sharing it would violate ownership and lifecycle boundaries.

## Data Pipeline
The CommandRegistry acts as a gatekeeper and context-enricher in the command registration data flow.

> Flow:
> Plugin Code (new MyCommand()) -> **CommandRegistry.registerCommand()** -> Global CommandManager -> Server Command Map

