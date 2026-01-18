---
description: Architectural reference for JavaPluginInit
---

# JavaPluginInit

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class JavaPluginInit extends PluginInit {
```

## Architecture & Concepts
The JavaPluginInit class is a specialized data carrier that encapsulates the complete initialization context for a single, dynamically loaded Java-based plugin. It serves as an immutable record, providing the core plugin system with all necessary information to bootstrap a plugin discovered on the filesystem, typically from a JAR file.

Its primary architectural role is to formalize the contract between the plugin discovery phase and the plugin loading phase. By bundling the plugin's manifest, its dedicated data directory, the path to its archive file, and its isolated class loader, it ensures that the PluginManager receives a consistent and self-contained unit of work.

The inclusion of a dedicated PluginClassLoader is the most critical aspect of this class. It signifies that the associated plugin operates within a sandboxed environment, isolated from the main server classpath and the classpaths of other plugins. This is a cornerstone of the server's stability and security model, preventing dependency conflicts and enabling plugins to be loaded and unloaded at runtime without corrupting the server's state.

### Lifecycle & Ownership
-   **Creation:** An instance of JavaPluginInit is created exclusively by the internal PluginLoader service during the server's startup or a runtime plugin-scan operation. It is instantiated immediately after a potential plugin JAR is identified and its manifest has been successfully parsed.
-   **Scope:** The object's lifetime is ephemeral and strictly bound to the loading process of a single plugin. It exists only to be passed from the PluginLoader to the PluginManager.
-   **Destruction:** Once the PluginManager has consumed the context to either successfully load the plugin or register a loading failure, the JavaPluginInit instance is no longer referenced and becomes eligible for garbage collection. It holds no resources that require explicit cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. All fields within this class and its parent are declared final and are populated exclusively through the constructor. This design guarantees that a plugin's initialization context is atomic and cannot be mutated after its creation, which is critical for predictable and safe plugin loading.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a JavaPluginInit object can be safely passed between threads without any external synchronization. This allows the plugin discovery and loading processes to be parallelized if the engine architecture requires it.

## API Surface
The public API is composed of simple accessors for retrieving the immutable context data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFile() | Path | O(1) | Retrieves the absolute filesystem path to the plugin's source JAR file. |
| getClassLoader() | PluginClassLoader | O(1) | Returns the dedicated, isolated class loader created for this specific plugin. |
| isInServerClassPath() | boolean | O(1) | Checks if the plugin's class loader is configured to share the main server classpath. |

## Integration Patterns

### Standard Usage
This class is an internal component of the plugin framework and is not intended for direct use by plugin developers. The following example illustrates its intended use within the server engine.

```java
// Example from within the server's PluginLoader service
Path pluginJarPath = Paths.get("plugins/my-plugin.jar");
PluginManifest manifest = parseManifest(pluginJarPath);
Path dataDirectory = createDataDirectory(manifest.getId());
PluginClassLoader loader = new PluginClassLoader(pluginJarPath, serverClassLoader);

// The context is created and passed to the manager for processing
PluginInit initContext = new JavaPluginInit(manifest, dataDirectory, pluginJarPath, loader);
pluginManager.loadPlugin(initContext);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Plugin code must never create an instance of JavaPluginInit. The plugin lifecycle is managed entirely by the server, and attempting to create this object manually will have no effect and indicates a misunderstanding of the plugin API.
-   **ClassLoader Misuse:** Do not retrieve the PluginClassLoader from a context object to attempt to bypass class loading isolation or inspect other plugins. This violates the sandboxing principle and can lead to severe runtime instability.
-   **State Modification:** Attempting to alter the fields of a JavaPluginInit instance using reflection is strictly forbidden. This would break the immutability guarantee and corrupt the plugin loading process.

## Data Pipeline
JavaPluginInit acts as a structured data container that carries information from the discovery stage to the execution stage of the plugin pipeline.

> Flow:
> Filesystem Scan (discovers plugin.jar) -> PluginLoader (parses manifest, creates classloader) -> **JavaPluginInit** (bundles context) -> PluginManager (consumes context) -> Active Plugin Instance

