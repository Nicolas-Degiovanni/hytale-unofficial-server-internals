---
description: Architectural reference for PluginBase
---

# PluginBase

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Managed Base Class

## Definition
```java
// Signature
public abstract class PluginBase implements CommandOwner {
```

## Architecture & Concepts
The PluginBase class is the foundational component for all server-side plugins and addons within the Hytale server architecture. It is not intended for direct instantiation but serves as the abstract parent for concrete plugin implementations, such as JavaPlugin.

Its primary architectural role is to provide a sandboxed, lifecycle-aware environment for each loaded plugin. This is achieved through a **Registry-per-Plugin** model. Each instance of a PluginBase derivative owns a complete set of service registries (CommandRegistry, EventRegistry, TaskRegistry, etc.). This design enforces strong isolation, preventing one plugin from directly interfering with the registrations of another.

PluginBase acts as the central service locator and facade for a plugin's interaction with core server systems. It manages a strict state machine that governs the plugin's lifecycle, ensuring that resources are registered and de-registered in a predictable and safe order. Any attempt to access services outside of the valid lifecycle states will result in a runtime exception, enforcing robust development patterns.

## Lifecycle & Ownership
The lifecycle of a PluginBase instance is strictly controlled by the server's central PluginManager and is not managed by the plugin developer.

-   **Creation:** An instance is created by the PluginManager during the server's initial boot sequence or when a plugin is loaded dynamically. The constructor is passed a PluginInit context object containing essential metadata like the plugin manifest and data directory. Direct instantiation by developers is unsupported and will fail.

-   **Scope:** The object persists for the entire duration that the plugin is considered "loaded" by the server. In a standard server session, this means it lives from server start to server shutdown.

-   **Destruction:** The PluginManager invokes the shutdown0 method during server shutdown or plugin unloading. This initiates a critical cleanup sequence:
    1.  The plugin's state is transitioned to SHUTDOWN.
    2.  The developer-overridden shutdown method is called for custom cleanup logic.
    3.  The internal cleanup method is invoked, which systematically shuts down every registry owned by the plugin (commands, events, tasks, etc.).
    4.  A list of registered shutdown tasks is executed in reverse order of registration, ensuring a clean, dependency-aware teardown.

## Internal State & Concurrency
-   **State:** PluginBase is a highly stateful object. Its core mutable state is the PluginState enum (e.g., NONE, SETUP, START, ENABLED, SHUTDOWN), which dictates its operational status. It also maintains mutable collections of configurations, shutdown hooks, and dynamically created codec registries.

-   **Thread Safety:** The class employs a mixed concurrency model.
    -   **Lifecycle methods** such as setup0, start0, and shutdown0 are **not thread-safe** and are designed to be called exclusively by the main server thread that manages the plugin system.
    -   **Internal collections** are chosen for thread safety where appropriate. The lists for configs and shutdownTasks use CopyOnWriteArrayList, and the map for codecMapRegistries uses ConcurrentHashMap. This allows for safe, concurrent registration of resources *after* the plugin has reached the ENABLED state.
    -   **WARNING:** While the container is thread-safe, the individual registries obtained from PluginBase may have their own specific threading requirements. Developers must assume that interactions with registries (e.g., CommandRegistry) should occur on the main server thread unless the specific registry's documentation states otherwise.

## API Surface
The public API provides access to the plugin's isolated registries and lifecycle hooks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | void | - | Override hook for initialization logic. Called once before start. |
| start() | void | - | Override hook for activation logic. Called after setup. |
| shutdown() | void | - | Override hook for de-initialization logic. Called on server shutdown. |
| getLogger() | HytaleLogger | O(1) | Retrieves the dedicated, pre-configured logger for this plugin. |
| getCommandRegistry() | CommandRegistry | O(1) | Gets the plugin-specific command registry. Throws if plugin is disabled. |
| getEventRegistry() | EventRegistry | O(1) | Gets the plugin-specific event registry. Throws if plugin is disabled. |
| getTaskRegistry() | TaskRegistry | O(1) | Gets the plugin-specific task scheduler. Throws if plugin is disabled. |
| withConfig(name, codec) | Config | O(1) | Declares a configuration file. Must be called in the constructor. |
| getState() | PluginState | O(1) | Returns the current lifecycle state of the plugin. |

## Integration Patterns

### Standard Usage
The standard pattern is to extend PluginBase (typically via JavaPlugin) and override the lifecycle methods to register game logic with the provided registries.

```java
// In a class extending JavaPlugin

@Override
public void start() {
    // 1. Get the plugin's isolated command registry
    CommandRegistry commands = this.getCommandRegistry();

    // 2. Register a new command
    commands.register(
        Command.builder("hello")
            .executes(ctx -> {
                ctx.getSource().sendMessage("Hello from my plugin!");
                return 1;
            })
            .build()
    );

    // 3. Register an event listener
    this.getEventRegistry().register(PlayerJoinEvent.class, event -> {
        getLogger().info("Player joined: " + event.getPlayer().getName());
    });
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new MyPlugin()`. The server's PluginManager is solely responsible for creating and managing the lifecycle of plugin instances.
-   **Registration Outside Lifecycle:** Do not attempt to register commands, events, or tasks within the plugin's constructor. Registries are not active until the setup and start phases. The internal state checks will throw an IllegalStateException.
-   **Storing Registries Statically:** Caching a registry in a static field can cause severe memory leaks and unpredictable behavior, especially in environments that support hot-reloading plugins. Always retrieve registries from the `this` context.
-   **Ignoring the State Machine:** Do not call lifecycle methods like `setup()` or `start()` directly. The server calls the non-overridable `setup0` and `start0` methods, which manage state and then delegate to the overridable methods.

## Data Pipeline
PluginBase itself is not a data processing component, but rather a controller that wires a plugin's logic into the server's various data and event streams.

An example flow for a registered command:
> Flow:
> Player Input -> Server Network Layer -> Command Dispatcher -> **PluginBase.getCommandRegistry()** -> Plugin-specific Command Executor

An example flow for a registered event listener:
> Flow:
> Game Logic (e.g., Entity Spawns) -> Global EventBus -> **PluginBase.getEventRegistry()** -> Plugin-specific Event Handler Method

