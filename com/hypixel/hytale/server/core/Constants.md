---
description: Architectural reference for Constants
---

# Constants

**Package:** com.hypixel.hytale.server.core
**Type:** Utility

## Definition
```java
// Signature
public final class Constants {
```

## Architecture & Concepts
The Constants class is the server's central, immutable repository for startup configuration. It serves as the primary bridge between external launch parameters, such as command-line arguments and file system state, and the internal logic of the server application.

Its most critical architectural function is to define the server's core composition via the **CORE_PLUGINS** manifest. This static array dictates which modules are loaded and, implicitly, their initialization order. This makes Constants the definitive source of truth for the server's fundamental architecture and feature set before any game logic executes.

Values within this class are resolved only once during JVM class loading. This design ensures that configuration is consistent and predictable throughout the server's entire lifecycle, eliminating the possibility of runtime configuration drift.

## Lifecycle & Ownership
- **Creation:** The Constants class is never instantiated. Its static fields are initialized by the JVM class loader the first time the class is referenced, which occurs very early in the server's bootstrap sequence. This initialization process involves parsing command-line options and performing initial file system checks.
- **Scope:** The state defined in Constants is global and persists for the entire lifetime of the server application.
- **Destruction:** The class and its static state are unloaded only when the Java Virtual Machine terminates. No explicit cleanup is required or possible.

## Internal State & Concurrency
- **State:** The state is **deeply immutable**. All public fields are declared static and final. Their values are computed once during the static initialization block and cannot be changed thereafter. This guarantees that configuration values are stable and reliable across all server systems.
- **Thread Safety:** This class is inherently **thread-safe**. The Java Memory Model guarantees that all final fields are safely and visibly published to all threads once the static initializer completes. Any system can read these constants from any thread without requiring locks or other synchronization primitives.

## API Surface
The public API consists exclusively of static final fields. The empty `init()` method exists as a potential hook for the bootstrap sequence but is currently a no-operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DEBUG | boolean | O(1) | Hardcoded flag indicating if the server is in a debug build. |
| SINGLEPLAYER | boolean | O(1) | True if the server was launched with the singleplayer flag. |
| ALLOWS_SELF_OP_COMMAND | boolean | O(1) | True if the server was launched with the flag to permit self-op commands. |
| FRESH_UNIVERSE | boolean | O(N) | True if the universe directory appears to be new or incomplete. Involves file system I/O on first access. |
| FORCE_NETWORK_FLUSH | boolean | O(1) | True if the server should aggressively flush network packets. |
| UNIVERSE_PATH | Path | O(1) | The resolved file system path to the server's universe directory. |
| CORE_PLUGINS | PluginManifest[] | O(1) | The definitive manifest of all core server modules to be loaded. This is the blueprint for the server's runtime composition. |

## Integration Patterns

### Standard Usage
Constants are designed for direct, static access from any part of the server codebase that requires startup configuration. The most common use cases are checking launch flags and bootstrapping the plugin system.

```java
// Checking a launch flag to alter game logic
if (Constants.SINGLEPLAYER) {
    // Apply singleplayer-specific rules
}

// The PluginLoader uses the manifest to build the server
for (PluginManifest manifest : Constants.CORE_PLUGINS) {
    pluginLoader.load(manifest);
}
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Do not attempt to modify the fields of this class using reflection. The server's behavior is undefined if these values change after initialization, and this will lead to severe instability.
- **Re-evaluation:** Do not write code that assumes these values can change. A check like `if (Constants.FRESH_UNIVERSE)` will yield the same result for the entire server session.
- **Dependency:** Do not make game logic systems tightly coupled to specific pathing logic derived from `UNIVERSE_PATH`. Instead, those systems should receive a `Path` object from a higher-level manager that uses `UNIVERSE_PATH` as its source.

## Data Pipeline
Constants acts as the origination point for configuration data, not as a processing stage. It translates external inputs into a static, in-memory representation for consumption by the rest of the application.

> Flow:
> Command Line Arguments & File System -> `joptsimple` & `java.nio` -> **Constants Static Initializer** -> All Server Systems (Plugin Loader, Universe, Network Manager)

