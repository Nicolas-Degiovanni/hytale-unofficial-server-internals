---
description: Architectural reference for PluginManager
---

# PluginManager

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Singleton

## Definition
```java
// Signature
public class PluginManager {
```

## Architecture & Concepts

The PluginManager is the central nervous system for the Hytale server's modular content, commonly known as plugins or mods. It is a foundational service responsible for the entire lifecycle of all plugins, from initial discovery and class loading to runtime management and graceful shutdown. Its architecture is designed around principles of isolation, dependency management, and a strict state-based lifecycle.

Key architectural pillars include:

*   **Plugin Discovery:** The manager systematically scans predefined directories (e.g., *mods*, *builtin*) and the application classpath for plugin archives (JAR files). It identifies potential plugins by locating a mandatory `manifest.json` file within each archive.

*   **Dependency Resolution & Load Order:** Before loading any code, the PluginManager builds a dependency graph based on the `dependencies` and `optionalDependencies` declared in each plugin's manifest. It performs a topological sort on this graph to determine a safe load order, ensuring that dependencies are always initialized before the plugins that require them. This process is critical for server stability and prevents cascading failures.

*   **Isolated Class Loading:** Each Java-based plugin is loaded into its own dedicated `PluginClassLoader`. This is a cornerstone of the architecture, providing strong isolation between plugins. It prevents common Java library conflicts (e.g., two plugins shipping different versions of Guava) and is the key enabling technology for dynamic unloading and reloading.

*   **Bridged Class Visibility:** While plugins are isolated, they must communicate. The `PluginBridgeClassLoader` acts as a controlled intermediary. When a plugin attempts to load a class, the bridge loader intercepts the request and attempts to find the class within another plugin, but *only* if a dependency is explicitly declared in the manifest. This enforces a strict "need-to-know" policy for inter-plugin communication.

*   **State-Driven Lifecycle:** The manager enforces a rigid lifecycle on all plugins, tracked by the `PluginState` enum (NONE -> SETUP -> START -> ENABLED -> SHUTDOWN). It orchestrates the transitions between these states, calling corresponding methods on each plugin instance (`setup0`, `start0`, `shutdown0`). Any deviation from this sequence results in a disabled plugin or a server fault.

## Lifecycle & Ownership

-   **Creation:** The PluginManager is a core singleton, instantiated once by the `HytaleServer` during its initial bootstrap sequence. The static `instance` field is set within the constructor, making it globally accessible via the static `get` method.

-   **Scope:** The PluginManager instance persists for the entire lifetime of the server process. Its state represents the complete picture of all loaded and available modular content.

-   **Destruction:** The `shutdown` method is invoked as part of the server's shutdown sequence. This method orchestrates the graceful shutdown of all active plugins in the reverse of their load order. The PluginManager object itself is eligible for garbage collection only when the server JVM exits.

## Internal State & Concurrency

-   **State:** The PluginManager is highly stateful and mutable. Its core state includes a registry of loaded plugins (`plugins`), a map of plugin file paths to their dedicated class loaders (`classLoaders`), and the current lifecycle state of the manager itself (`state`). This internal state is considered a source of truth for the server's plugin configuration.

-   **Thread Safety:** This class is designed to be thread-safe, but its operations can be contentious. A central `ReentrantReadWriteLock` protects the primary plugin registry (`plugins`) from concurrent modification.
    -   Read operations like `getPlugin` acquire a read lock, allowing for high-concurrency lookups.
    -   Write operations that modify the plugin landscape—`setup`, `start`, `shutdown`, `load`, `unload`—acquire an exclusive write lock, serializing these critical state changes.

    **WARNING:** A critical and complex locking interaction exists with the asset system. The `unload` method acquires both the PluginManager's write lock and the `AssetRegistry.ASSET_LOCK`. This two-phase locking is essential to prevent race conditions where a plugin is unloaded while its assets are still being accessed or registered. Developers extending this system must be acutely aware of this locking hierarchy to avoid deadlocks.

## API Surface

The public API exposes high-level control over the entire plugin system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | PluginManager | O(1) | Static accessor for the singleton instance. |
| setup() | void | O(N log N + I/O) | Discovers all plugins, resolves dependencies, and transitions them to the SETUP state. Must be called once before `start`. |
| start() | void | O(N) | Transitions all setup plugins to the ENABLED state, making them fully active. Must be called after `setup`. |
| shutdown() | void | O(N) | Gracefully shuts down and disables all loaded plugins. Called during server shutdown. |
| load(identifier) | boolean | O(I/O) | Dynamically loads, sets up, and starts a single, previously unloaded plugin at runtime. |
| unload(identifier) | boolean | O(N) | Dynamically shuts down and unloads a plugin, closing its ClassLoader and releasing its resources. This is a highly complex operation. |
| reload(identifier) | boolean | O(unload + load) | Convenience method to perform an atomic unload and subsequent load of a plugin. |
| getPlugins() | List<PluginBase> | O(N) | Returns a copy of the list of all currently loaded and active plugins. |

## Integration Patterns

### Standard Usage

Direct interaction with the PluginManager is typically reserved for server commands or administrative tools. The server core manages the initial `setup` and `start` calls.

```java
// Example: A server command to reload a specific plugin
PluginManager manager = PluginManager.get();
PluginIdentifier targetId = new PluginIdentifier("com.example", "myplugin");

// This operation is atomic and thread-safe
boolean success = manager.reload(targetId);

if (success) {
    System.out.println("Plugin " + targetId + " reloaded successfully.");
} else {
    System.out.println("Failed to reload plugin " + targetId + ". It may not be loaded or a failure occurred.");
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new PluginManager()`. It is a singleton managed exclusively by the server's core bootstrap process. Always use the static `PluginManager.get()` method.

-   **Lifecycle Violation:** Calling lifecycle methods out of order (e.g., `start()` before `setup()`) will throw an `IllegalStateException` and destabilize the server. The state machine must be respected.

-   **Modifying Internal Collections:** Do not attempt to use reflection to modify the internal `plugins` or `classLoaders` maps. All modifications must go through the public API (`load`, `unload`) to ensure locks are acquired and state transitions are handled correctly.

-   **Ignoring Return Values:** Methods like `load`, `unload`, and `reload` return a boolean indicating success. Ignoring this value can lead to incorrect assumptions about the state of a plugin. A `false` return value indicates that the desired state change did not complete and the plugin may be in a disabled or partially unloaded state.

## Data Pipeline

The process of loading a single Java plugin from a file on disk follows a well-defined pipeline.

> Flow:
> File System Scan (`mods/*.jar`) -> JAR Manifest Read (`manifest.json`) -> **PluginManager** (creates `PendingLoadJavaPlugin`) -> Dependency Validation & Topological Sort -> **PluginManager** (instantiates `PluginClassLoader`) -> Reflection (instantiates `PluginBase` implementation) -> Lifecycle Invocation (`setup0`, `start0`) -> Final Registration (added to active `plugins` map)

