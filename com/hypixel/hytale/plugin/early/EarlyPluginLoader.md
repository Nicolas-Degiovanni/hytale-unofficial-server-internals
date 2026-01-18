---
description: Architectural reference for EarlyPluginLoader
---

# EarlyPluginLoader

**Package:** com.hypixel.hytale.plugin.early
**Type:** Static Utility

## Definition
```java
// Signature
public final class EarlyPluginLoader {
```

## Architecture & Concepts
The EarlyPluginLoader is a foundational component of the application bootstrap sequence, responsible for enabling low-level modifications to the engine's core Java classes. It operates entirely outside the standard, sandboxed plugin system.

Its primary purpose is to discover and load special JAR files containing **ClassTransformer** implementations before the main application initializes. This allows for Java bytecode instrumentation at the earliest possible stage, a technique required by advanced modding frameworks to alter fundamental engine behaviors.

The loader identifies plugins from two sources: a default **earlyplugins** directory and additional paths specified via command-line arguments. It uses a dedicated URLClassLoader to isolate these plugins and discovers transformers via the standard Java ServiceLoader mechanism.

This system is inherently powerful and dangerous. It provides a mechanism to bypass engine-level abstractions and directly manipulate class bytecode. Consequently, its use is officially unsupported and triggers explicit warnings to the user, requiring interactive confirmation to proceed unless a specific override flag is provided. The loader itself does not perform the transformation; it prepares the list of transformers for a consumer, typically a Java Instrumentation Agent, which executes the bytecode modifications.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a final class with a private constructor, it exposes only static methods and manages global static state.
- **Scope:** The state managed by EarlyPluginLoader, specifically the list of transformers and the plugin class loader, is global and application-scoped. This state is initialized once by the `loadEarlyPlugins` method during the initial moments of application startup and persists for the entire lifetime of the JVM process.
- **Destruction:** State is not explicitly destroyed or reset. The ClassLoader and all loaded plugin classes are held in memory until the JVM terminates.

## Internal State & Concurrency
- **State:** The class manages mutable, static state. The internal list of transformers and the URLClassLoader instance are populated once during the `loadEarlyPlugins` call. While the list itself is mutable during initialization, it is exposed to consumers as an unmodifiable view to prevent external changes.
- **Thread Safety:** This class is **not thread-safe** and is fundamentally designed for single-threaded execution during application startup.
    - **WARNING:** Invoking `loadEarlyPlugins` from multiple threads will result in severe race conditions, leading to a corrupted internal state, resource leaks, or application instability.
    - All public accessor methods like `getTransformers` are safe to call from any thread *only after* the `loadEarlyPlugins` method has fully completed its execution on the main thread.

## API Surface
The public API provides the entry point for initialization and read-only access to the resulting state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadEarlyPlugins(args) | void | O(N) | Initializes the system. Scans for plugins, loads transformers, and handles user consent. Throws IOExceptions on file system errors. **Must only be called once.** |
| hasTransformers() | boolean | O(1) | Returns true if one or more ClassTransformer instances were successfully discovered and loaded. |
| getTransformers() | List | O(1) | Returns an unmodifiable view of the loaded transformers, sorted in descending order of priority. |
| getPluginClassLoader() | URLClassLoader | O(1) | Returns the dedicated class loader for early plugins. Returns null if no plugins were loaded. |

## Integration Patterns

### Standard Usage
The EarlyPluginLoader must be invoked at the very beginning of the application's main entry point, before any other significant engine systems are initialized. A separate system, such as a Java Agent, then consumes the loaded transformers.

```java
// In the primary application entry point (e.g., main method)
public static void main(String[] args) {
    // Load transformers before anything else.
    EarlyPluginLoader.loadEarlyPlugins(args);

    // A separate instrumentation system would then retrieve the transformers.
    if (EarlyPluginLoader.hasTransformers()) {
        List<ClassTransformer> transformers = EarlyPluginLoader.getTransformers();
        
        // Pass transformers to the bytecode instrumentation agent...
        InstrumentationService.getInstance().register(transformers);
    }

    // ... continue with normal application bootstrap.
}
```

### Anti-Patterns (Do NOT do this)
- **Multiple Invocations:** Never call `loadEarlyPlugins` more than once. Doing so will attempt to re-create the class loader and re-scan for plugins, leading to unpredictable behavior and resource leaks.
- **Late Invocation:** Calling `loadEarlyPlugins` after core game classes have already been loaded by the JVM is useless. The purpose of this loader is to enable transformation *before* classes are defined. A late call will have no effect.
- **State Modification:** Do not attempt to cast or use reflection to modify the list returned by `getTransformers`. It is explicitly unmodifiable to guarantee state integrity after initialization.

## Data Pipeline
The flow of data is unidirectional, starting from the environment and resulting in a list of transformers ready for consumption.

> Flow:
> Command-Line Arguments & Filesystem -> **EarlyPluginLoader.loadEarlyPlugins** -> URLClassLoader Creation -> ServiceLoader Discovery -> Internal Transformer List -> Consumer (e.g., Instrumentation Agent)

