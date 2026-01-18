---
description: Architectural reference for PendingLoadPlugin
---

# PendingLoadPlugin

**Package:** com.hypixel.hytale.server.core.plugin.pending
**Type:** Abstract State Object

## Definition
```java
// Signature
public abstract class PendingLoadPlugin {
```

## Architecture & Concepts
The PendingLoadPlugin class is a foundational component of the server's **Plugin Loading Subsystem**. It serves as an intermediate, in-memory representation of a plugin that has been discovered but not yet fully loaded or initialized.

This class acts as a "blueprint" or "promise" for a future plugin instance. It decouples the initial *discovery* phase (scanning files, reading manifests) from the complex *instantiation* phase (class loading, dependency injection, lifecycle management). By abstracting a potential plugin into a PendingLoadPlugin, the system can gather all available plugins, analyze their relationships, and resolve a safe execution order before committing to loading any code.

The most critical architectural function is provided by the static **calculateLoadOrder** method. This method implements a topological sort on the graph of all discovered plugins and their dependencies. It is the central authority for dependency resolution, ensuring that plugins are loaded in an order that respects their explicit dependencies, optional dependencies, and load-before directives. This preemptively solves entire classes of runtime errors, such as missing dependency exceptions and cyclic dependency deadlocks.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated by a higher-level **Plugin Discovery Service** during server bootstrap. This service scans the classpath and designated plugin directories, parses each plugin's manifest file, and creates a corresponding PendingLoadPlugin object for each valid manifest.
- **Scope:** These objects are ephemeral and exist only during the server's startup sequence. Their entire lifecycle is confined to the plugin loading and resolution phase.
- **Destruction:** After the **calculateLoadOrder** method successfully produces an ordered list, the system iterates through it, calling **load** on each object to create the final **PluginBase** instance. Once this process is complete, the entire collection of PendingLoadPlugin objects is no longer referenced and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** An instance of PendingLoadPlugin is effectively **immutable**. Its core state, including the identifier, manifest, and path, is set at construction and does not change. This immutability is crucial for predictable dependency resolution.
- **Thread Safety:** The class is **not thread-safe**. While individual instances are immutable, the static **calculateLoadOrder** method is stateful during its execution, internally mutating a graph representation of the plugins. It is fundamentally designed for synchronous, single-threaded execution during server startup.

**WARNING:** Calling **calculateLoadOrder** from multiple threads or modifying the input map concurrently will lead to race conditions, incorrect load orders, or unrecoverable runtime exceptions. This entire subsystem must be treated as a single-threaded bootstrap process.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | PluginBase | High | Abstract method. Loads the plugin code into memory and returns an initialized PluginBase instance. Involves I/O and class loading. |
| createSubPendingLoadPlugins() | List<PendingLoadPlugin> | O(N) | Creates pending load representations for all sub-plugins defined in the manifest. N is the number of sub-plugins. |
| dependsOn(identifier) | boolean | O(1) | Checks if this plugin declares a dependency on the given identifier. |
| isInServerClassPath() | boolean | O(1) | Abstract method. Determines if the plugin is an internal component versus a third-party file. Critical for dependency sorting. |
| calculateLoadOrder(pending) | static List<PendingLoadPlugin> | O(V+E) | Performs a topological sort on the entire set of discovered plugins. V is the number of plugins, E is the number of dependency edges. Throws IllegalArgumentException on failure. |

## Integration Patterns

### Standard Usage
The intended consumer of this class is the server's primary plugin management service. The standard operational flow is to first discover all plugins, then calculate the load order, and finally load them sequentially.

```java
// 1. Discover all plugins from various sources
Map<PluginIdentifier, PendingLoadPlugin> discoveredPlugins = pluginDiscoveryService.findAll();

// 2. Calculate the safe loading order
// This is the primary entry point for using the system.
List<PendingLoadPlugin> loadOrder;
try {
    loadOrder = PendingLoadPlugin.calculateLoadOrder(discoveredPlugins);
} catch (IllegalArgumentException e) {
    // Handle missing dependencies or cyclic dependency errors
    server.getLogger().fatal("Failed to resolve plugin dependencies!", e);
    return;
}

// 3. Load each plugin in the calculated order
for (PendingLoadPlugin pendingPlugin : loadOrder) {
    PluginBase loadedPlugin = pendingPlugin.load();
    pluginRegistry.register(loadedPlugin);
}
```

### Anti-Patterns (Do NOT do this)
- **Bypassing the Sorter:** Do not iterate over a raw collection of PendingLoadPlugin objects and call **load** on them directly. This completely bypasses the dependency resolution logic and will almost certainly fail in any non-trivial environment.
- **Ignoring Exceptions:** The **calculateLoadOrder** method throws an **IllegalArgumentException** for critical, unrecoverable errors like missing or circular dependencies. This exception must be caught and should terminate the server startup process. Ignoring it will leave the server in a corrupt, unstable state.
- **Manual Instantiation:** Do not attempt to manually construct PendingLoadPlugin instances. They should only be created by the official discovery mechanism to ensure manifest data is parsed and validated correctly.

## Data Pipeline
The PendingLoadPlugin class is a critical stage in the data pipeline that transforms plugin artifacts on disk into running code within the server.

> Flow:
> Plugin Artifacts (JARs on Filesystem) -> Plugin Discovery Service -> **Map of PendingLoadPlugin Instances** -> **calculateLoadOrder** -> Ordered List of PendingLoadPlugin -> Iterative **load()** calls -> Initialized PluginBase Instances -> Plugin Registry

