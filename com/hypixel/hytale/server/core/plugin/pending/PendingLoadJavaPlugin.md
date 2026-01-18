---
description: Architectural reference for PendingLoadJavaPlugin
---

# PendingLoadJavaPlugin

**Package:** com.hypixel.hytale.server.core.plugin.pending
**Type:** Transient Factory

## Definition
```java
// Signature
public class PendingLoadJavaPlugin extends PendingLoadPlugin {
```

## Architecture & Concepts
The PendingLoadJavaPlugin class is a concrete implementation of the plugin loading strategy, representing a Java-based plugin that has been discovered but not yet fully instantiated. It serves as a critical intermediate step in the server's plugin lifecycle, bridging the gap between file-system discovery and runtime activation.

Its primary responsibility is to act as a stateful factory. It encapsulates all necessary context for a plugin's final instantiation: its manifest, its dedicated PluginClassLoader, and its source file path. This encapsulation ensures that the final loading process is self-contained and isolated.

The use of a dedicated PluginClassLoader is central to the design. This allows for robust plugin isolation, preventing dependency conflicts between different plugins or between a plugin and the core server. This class is the final authority that uses reflection to construct the plugin's main class, thereby transforming a static definition on disk into a live, executable object within the server environment.

### Lifecycle & Ownership
- **Creation:** An instance is created by the PluginManager during the server's plugin discovery phase. This typically occurs when the manager scans the mods directory, finds a JAR file, and successfully parses its embedded plugin manifest.
- **Scope:** This object is short-lived and exists only for the duration of the plugin loading sequence. Its purpose is fulfilled the moment its `load` method is invoked.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection immediately after the PluginManager processes it. There is no explicit destruction or cleanup method, as it holds no persistent resources itself; resource ownership is managed by the contained PluginClassLoader.

## Internal State & Concurrency
- **State:** The object is effectively immutable. All its core fields, including the manifest and the PluginClassLoader, are final and are injected via the constructor. The `load` method is stateless concerning the PendingLoadJavaPlugin instance itself, though it manipulates the state of the JVM's class loading system.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent use. It is expected to be managed and operated by a single thread within the PluginManager's sequential loading logic. Calling the `load` method from multiple threads on the same instance will lead to unpredictable behavior in the underlying class loader and is a critical anti-pattern.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | JavaPlugin | Complex | Attempts to load, initialize, and instantiate the plugin. This is the primary operation. Returns the live JavaPlugin instance on success or null on any failure. |
| createSubPendingLoadPlugin(manifest) | PendingLoadPlugin | O(1) | Creates a new pending load operation for a sub-plugin, sharing the parent's class loader and file path. |
| isInServerClassPath() | boolean | O(1) | Delegates to the class loader to determine if its classes are isolated or exposed to the main server classpath. |

## Integration Patterns

### Standard Usage
This class is an internal component of the plugin system. A service like the PluginManager discovers plugin candidates and creates PendingLoadJavaPlugin instances, which it then processes to get live plugin objects.

```java
// Conceptual usage within a PluginManager
for (PendingLoadPlugin pending : discoveredPlugins) {
    if (pending instanceof PendingLoadJavaPlugin) {
        // The load method performs the critical instantiation step
        JavaPlugin plugin = ((PendingLoadJavaPlugin) pending).load();

        if (plugin != null) {
            pluginRegistry.register(plugin);
        } else {
            // Log and handle the failed plugin load
            logger.warn("Failed to activate plugin from: " + pending.getPath());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct this class manually. It is tightly coupled to the server's plugin discovery and class loading infrastructure and should only be created by the responsible service.
- **Reusing Instances:** Do not invoke the `load` method more than once. While it may appear to work, it bypasses the intended lifecycle and can cause unpredictable class loading side effects. Treat each instance as a single-use factory.
- **Concurrent Access:** Never share an instance across multiple threads. The class loading and reflection operations are not synchronized and are unsafe for concurrent execution.

## Data Pipeline
The PendingLoadJavaPlugin acts as the instantiation stage in the overall plugin loading pipeline.

> Flow:
> Plugin JAR on Disk -> PluginManager Discovery -> **PendingLoadJavaPlugin** (Encapsulation) -> `load()` Method (Reflection & Instantiation) -> Live JavaPlugin Object -> Plugin Registry

