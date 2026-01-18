---
description: Architectural reference for PluginInit
---

# PluginInit

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class PluginInit {
```

## Architecture & Concepts
The PluginInit class is a foundational data structure used during the server's plugin bootstrap sequence. It functions as a configuration and context object, encapsulating the essential information a plugin requires to initialize itself within the server environment.

This class acts as the formal contract between the server's Plugin Loading System and the plugin's entry point. It is constructed by the server core *before* the plugin's own code is executed, bundling two critical pieces of information:
1.  The parsed **PluginManifest**, containing metadata like the plugin's name, version, and main class.
2.  A dedicated **Path** to the plugin's data directory, ensuring sandboxed file system access.

The method isInServerClassPath, which currently returns a hardcoded true, signals that the system is designed for plugins bundled directly with the server's classpath. This has significant architectural implications, suggesting that dynamic, hot-swappable plugins loaded from arbitrary locations are not a primary design goal for this component.

### Lifecycle & Ownership
-   **Creation:** An instance of PluginInit is created exclusively by the server's internal PluginLoader during the server startup phase. For each discovered plugin, the loader parses its manifest and allocates a data directory, then uses this information to instantiate a unique PluginInit object.
-   **Scope:** This object is ephemeral and has a very short lifecycle. It exists only for the duration of a single plugin's initialization call (e.g., passed as a parameter to the plugin's constructor or an `onLoad` method).
-   **Destruction:** The object is eligible for garbage collection immediately after the plugin's initialization logic completes. Plugins are expected to extract and store the necessary data (like the data directory path) rather than holding a reference to the PluginInit object itself.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields are declared as final and are assigned only once during construction. This guarantees that the plugin's initial environment cannot be mutated after creation, providing stability during the critical startup process.
-   **Thread Safety:** **Fully thread-safe**. Its immutability ensures that it can be safely read by any thread without requiring locks or other synchronization primitives. This is vital in a multi-threaded server environment where plugin loading may be parallelized in the future.

## API Surface
The public API is minimal, consisting only of non-blocking data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPluginManifest() | PluginManifest | O(1) | Returns the parsed manifest metadata for the plugin. |
| getDataDirectory() | Path | O(1) | Returns the absolute path to the plugin's private data directory. |
| isInServerClassPath() | boolean | O(1) | Confirms if the plugin is loaded from the main server classpath. |

## Integration Patterns

### Standard Usage
The PluginInit object is provided by the server to the plugin's main class during its construction or a dedicated initialization method. The plugin should immediately query this object to configure its internal state.

```java
// Example inside a hypothetical Plugin main class
public class MyPlugin {

    private final Path configPath;
    private final Logger logger;

    // The server injects PluginInit during construction
    public MyPlugin(PluginInit init) {
        PluginManifest manifest = init.getPluginManifest();
        Path dataDirectory = init.getDataDirectory();

        // Use the provided context to set up the plugin
        this.logger = LoggerFactory.getLogger(manifest.getName());
        this.configPath = dataDirectory.resolve("config.yml");
        
        this.logger.info("Initializing " + manifest.getName() + " version " + manifest.getVersion());
        loadConfiguration();
    }

    private void loadConfiguration() {
        // ... logic to load files from this.configPath
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PluginInit()`. This object is a token from the server's core systems. Creating one manually will result in a plugin that is completely disconnected from the server's managed lifecycle and file system.
-   **Long-Term Retention:** Do not store the PluginInit object in a field for later use. Its purpose is fulfilled once initialization is complete. Extract the required values (like the Path) into your own fields and let the PluginInit object be garbage collected.

## Data Pipeline
PluginInit is not a data processor; it is a source of initial state. It represents the culmination of the server's discovery and setup process, which is then fed into the plugin to begin its own lifecycle.

> Flow:
> Server Boot -> Plugin Discovery -> Manifest Parsing -> **PluginInit (Instantiation)** -> Plugin Constructor Injection -> Plugin Initialization Logic

