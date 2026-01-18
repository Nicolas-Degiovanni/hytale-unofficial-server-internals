---
description: Architectural reference for JavaPlugin
---

# JavaPlugin

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Component (Abstract Base)

## Definition
```java
// Signature
public abstract class JavaPlugin extends PluginBase {
```

## Architecture & Concepts
The JavaPlugin class is the foundational component for all server-side plugins developed in Java. It serves as the primary bridge between the core server engine and dynamically loaded plugin code. Each loaded Java plugin is represented by a concrete instance of a JavaPlugin subclass.

Architecturally, this class enforces a critical security and stability boundary. Each JavaPlugin is isolated within its own dedicated PluginClassLoader. This sandboxing prevents plugins from interfering with the server's internal classes or the classes of other plugins, which is essential for a stable, multi-plugin environment.

A key responsibility of this class is resource integration. During its startup lifecycle, it inspects the plugin's manifest for an embedded asset pack. If one is declared, the JavaPlugin orchestrates its registration with the server's central AssetModule, making the plugin's custom models, textures, and other assets available to the game.

## Lifecycle & Ownership
The lifecycle of a JavaPlugin instance is strictly managed by the server's plugin framework and is not intended for manual control.

-   **Creation:** A concrete subclass of JavaPlugin is instantiated by the server's PluginManager during the plugin loading sequence. The constructor's dependency on a JavaPluginInit object, which is an internally managed context, effectively prohibits direct instantiation by plugin developers.
-   **Scope:** An instance of a JavaPlugin persists for the entire duration that the plugin is enabled. Its lifetime is coupled one-to-one with the loaded plugin.
-   **Destruction:** The instance is marked for garbage collection when the plugin is disabled or the server shuts down. The PluginManager is responsible for releasing all references and ensuring the associated PluginClassLoader can also be collected, which is critical for enabling clean reloads and preventing memory leaks.

## Internal State & Concurrency
-   **State:** The base state of JavaPlugin is effectively immutable after construction. The path to the plugin file and its dedicated class loader are final fields. State management and mutability are the responsibility of the concrete implementation that extends this class.
-   **Thread Safety:** This base class is not inherently thread-safe. All interactions with a JavaPlugin instance and its subclasses are expected to occur on the main server thread. The `start0` method interacts with the AssetModule, which is a thread-safe singleton, but any state within the plugin subclass must be synchronized if it is to be accessed from asynchronous tasks.

## API Surface
The public API provides essential, read-only context about the plugin's runtime environment.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFile() | Path | O(1) | Returns the absolute path to the plugin's source JAR or directory. |
| getClassLoader() | PluginClassLoader | O(1) | Provides access to the plugin's dedicated class loader. |
| getType() | PluginType | O(1) | Returns the type identifier for this plugin, always PLUGIN. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or interact with this class directly. The standard pattern is to extend it to create the main entry point for a new plugin.

```java
// In MyPlugin.java, which is the main class defined in plugin.json
public class MyPlugin extends JavaPlugin {

    // The server's PluginManager will invoke this constructor
    public MyPlugin(JavaPluginInit init) {
        super(init);
    }

    @Override
    public void onEnable() {
        // Plugin startup logic, such as registering event listeners
        // or scheduling tasks, goes here.
        getLogger().info("MyPlugin has been enabled successfully.");
    }

    @Override
    public void onDisable() {
        // Plugin shutdown and cleanup logic.
        getLogger().info("MyPlugin is shutting down.");
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Never attempt to create an instance of your plugin class using `new MyPlugin()`. The plugin lifecycle is exclusively managed by the server, and manual creation will result in a non-functional and uninitialized object.
-   **ClassLoader Tampering:** Do not attempt to modify the state of the PluginClassLoader obtained from getClassLoader. Relying on custom modifications can break class resolution and lead to severe server instability.
-   **Premature Asset Access:** Do not attempt to access assets defined in your plugin's asset pack from your plugin's constructor. Asset registration occurs later in the `start0` lifecycle phase, and assets will not be available until the `onEnable` method is called.

## Data Pipeline
The JavaPlugin acts as an initiator for the asset registration pipeline. It does not process continuous data streams but instead injects a new data source (the asset pack) into a core system during its initialization.

> Flow:
> Plugin JAR on Disk -> Server PluginManager -> **JavaPlugin** (during `start0` lifecycle) -> AssetModule.registerPack(id, file) -> Global Asset System Cache

