---
description: Architectural reference for PluginClassLoader
---

# PluginClassLoader

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Utility

## Definition
```java
// Signature
public class PluginClassLoader extends URLClassLoader {
```

## Architecture & Concepts
The PluginClassLoader is the cornerstone of the Hytale plugin security and isolation model. As a custom implementation of URLClassLoader, its primary responsibility is to load a plugin's classes and resources from its JAR file, while strictly controlling its access to the server's internal APIs and other plugins.

Unlike the standard Java parent-first delegation model, PluginClassLoader implements a custom, multi-stage search hierarchy. This non-standard approach is a critical design decision that enables plugin sandboxing. The class loading order is as follows:

1.  **Server ClassLoader:** It first attempts to resolve a class using the main server class loader (`PluginManager.class.getClassLoader()`). This ensures that plugins can access shared server APIs (like the event system or entity components) without bundling them, preventing version conflicts.
2.  **Self (Plugin JAR):** If the class is not a server class, it attempts to load it from its own repository of URLs, which point to the plugin's JAR file. This isolates the plugin's internal classes and dependencies.
3.  **Bridge ClassLoader:** As a final fallback, it delegates to the `PluginManager.PluginBridgeClassLoader`. This is a shared, intermediate class loader responsible for managing libraries or APIs intended for cross-plugin communication, preventing each plugin from loading its own copy of a shared dependency.

This class also distinguishes between trusted "Builtin" plugins and sandboxed "ThirdParty" plugins via the `inServerClassPath` flag. This distinction is embedded in the class loader's name, which is later used by the static `isFromThirdPartyPlugin` utility to identify the origin of exceptions for enhanced security and error reporting.

## Lifecycle & Ownership
-   **Creation:** A PluginClassLoader is instantiated exclusively by the `PluginManager` during the loading phase of a specific plugin. One instance is created for each plugin.
-   **Scope:** The lifecycle of a PluginClassLoader is directly coupled to the lifecycle of its associated `JavaPlugin`. It persists in memory for as long as the plugin is loaded and enabled.
-   **Destruction:** The class loader, along with all the classes it has loaded, becomes eligible for garbage collection when the `PluginManager` unloads the plugin. This is the mechanism that allows for dynamic unloading and reloading of plugins without a server restart.

## Internal State & Concurrency
-   **State:** The PluginClassLoader is stateful. It maintains a reference to the master `PluginManager`, the `JavaPlugin` instance it serves, and the URLs of the plugin's JAR file. It inherits the internal class cache from `URLClassLoader`, storing classes it has already loaded.
-   **Thread Safety:** The class is designed for concurrent operation. The static initializer calls `registerAsParallelCapable()`, signaling to the JVM that it can handle class loading requests from multiple threads simultaneously without external locking. Specific methods like `loadLocalClass` use finer-grained locking (`getClassLoadingLock`) to ensure atomicity for individual class loading operations.

## API Surface
The primary consumers of this API are the Java Virtual Machine itself and the parent `PluginManager`. Direct interaction by plugin developers is not intended or supported.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadClass(name, resolve) | Class<?> | O(N) | Overrides the default behavior to implement the Server -> Self -> Bridge loading strategy. Throws ClassNotFoundException if the class cannot be found in any of the valid locations. |
| loadLocalClass(name) | Class<?> | O(N) | A specialized variant to find and load a class primarily from the plugin's own JAR. |
| getResource(name) | URL | O(N) | Finds a resource, following the same Server -> Self -> Bridge search path as class loading. |
| getResources(name) | Enumeration<URL> | O(N) | Finds all occurrences of a resource across the Server, Self, and Bridge loaders, aggregating the results. |
| isFromThirdPartyPlugin(throwable) | static boolean | O(S*F) | A diagnostic utility to inspect an exception's stack trace to determine if code from a third-party plugin was involved. S is stack depth, F is frames per throwable. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The JVM's class loading mechanism and the server's `PluginManager` are the sole integrators. The system works transparently when a plugin's code executes.

```java
// This is an internal server operation, not for plugin developers.
// The PluginManager creates the loader for a given plugin file.
PluginClassLoader loader = new PluginClassLoader(
    this.pluginManager,
    false, // This is a third-party plugin
    pluginJarFile.toURI().toURL()
);

// The manager then uses this loader to instantiate the plugin's main class.
Class<?> mainClass = loader.loadClass("com.example.myplugin.Main");
JavaPlugin plugin = (JavaPlugin) mainClass.getConstructor().newInstance();

// The loader is then associated with the plugin instance.
loader.setPlugin(plugin);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create a `PluginClassLoader` with `new`. Its lifecycle is strictly managed by the `PluginManager` to ensure proper sandboxing and resource management.
-   **Assuming Parent-First Delegation:** Do not write code that relies on the standard Java class loading hierarchy. A plugin cannot override server classes, and its visibility into other plugins is strictly controlled by the `PluginBridgeClassLoader`.
-   **Reflective Access:** Attempting to use reflection to access or manipulate a plugin's class loader at runtime is a severe violation of the security model and will likely lead to a `SecurityException` or destabilize the server.

## Data Pipeline
PluginClassLoader does not process data in a traditional sense. Instead, it orchestrates a *class resolution pipeline*.

> Flow:
> JVM Request for Class -> **PluginClassLoader.loadClass** -> Attempt Server Loader -> Attempt Self (Plugin JAR) -> Attempt Bridge Loader -> Return `Class` Object or Throw `ClassNotFoundException`

