---
description: Architectural reference for SprintForcePlugin
---

# SprintForcePlugin

**Package:** com.hypixel.hytale.builtin.sprintforce
**Type:** Plugin

## Definition
```java
// Signature
public class SprintForcePlugin extends JavaPlugin {
```

## Architecture & Concepts
The SprintForcePlugin is a built-in server plugin responsible for declaring server-side support for a specific client feature: SprintForce. It follows a declarative, configuration-as-code pattern. The plugin itself contains no gameplay logic; its sole function is to register the SprintForce capability with the server's core systems during the startup sequence.

This registration modifies the server's behavior during the client-server handshake. By registering this feature, the server informs connecting clients that it understands and supports the sprint force mechanic. The actual implementation of the movement logic is handled by other core engine systems, which are conditionally activated based on the presence of this feature in the ClientFeatureRegistry.

This component acts as a modular switch, allowing server operators to enable or disable the feature simply by including or excluding the plugin from the server's runtime.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central PluginManager during the server bootstrap phase. The PluginManager scans for available plugins, creates an instance of each, and provides a JavaPluginInit context object required by the constructor.
- **Scope:** The SprintForcePlugin instance is a session-scoped object. It persists for the entire duration of the server's runtime.
- **Destruction:** The object is marked for garbage collection upon server shutdown or if the plugin is explicitly unloaded by an administrator via server commands.

## Internal State & Concurrency
- **State:** This class is stateless. It does not define any of its own member variables and does not cache data. Its purpose is to perform a one-time, fire-and-forget operation that modifies the state of a separate, shared system (the ClientFeatureRegistry).
- **Thread Safety:** The instance is inherently thread-safe due to its stateless nature. The `setup` method is invoked by the PluginManager on the main server thread during initialization, a phase that is not concurrent. Subsequent interaction with the object is not expected.

## API Surface
The public contract is defined by the JavaPlugin lifecycle, not for general consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Lifecycle callback invoked by the PluginManager. Registers the ClientFeature.SprintForce enum with the server's ClientFeatureRegistry. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by other code. The server's plugin loading mechanism is the sole integrator. Its presence in the server's plugin directory is sufficient to activate its behavior.

```java
// The server's PluginManager handles this internally during startup
// This code is conceptual and not intended for developers to write.

PluginManager manager = server.getPluginManager();
JavaPluginInit initContext = manager.createInitContextFor(SprintForcePlugin.class);
SprintForcePlugin plugin = new SprintForcePlugin(initContext);

manager.registerAndEnable(plugin); // This will trigger the setup() call
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SprintForcePlugin()`. The plugin requires a valid context from the PluginManager to function. A manually created instance would be inert and serve no purpose.
- **Manual Invocation:** Do not call the `setup` method directly. It is a lifecycle callback managed by the plugin system. Calling it outside of the server's initialization sequence can lead to race conditions or throw exceptions if the ClientFeatureRegistry is not in a writable state.

## Data Pipeline
This component does not process a continuous stream of data. Instead, it participates in the server's startup configuration pipeline.

> Flow:
> Server Bootstrap -> PluginManager Scan -> **SprintForcePlugin.setup()** -> ClientFeatureRegistry Update -> Modified Server-to-Client Handshake Data

