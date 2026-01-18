---
description: Architectural reference for MantlingPlugin
---

# MantlingPlugin

**Package:** com.hypixel.hytale.builtin.mantling
**Type:** Managed Singleton (per-plugin context)

## Definition
```java
// Signature
public class MantlingPlugin extends JavaPlugin {
```

## Architecture & Concepts
The MantlingPlugin is a server-side plugin component responsible for feature declaration. Its sole purpose is to register the **Mantling** game mechanic with the server's core systems during the bootstrap phase.

This class is an implementation of the Plugin pattern. It does not contain any active game logic itself. Instead, it acts as a manifest entry, signaling to the server and connecting clients that the mantling feature is available and enabled. The server's `PluginLoader` discovers and instantiates this class, integrating it into the server lifecycle. The actual implementation of the mantling mechanic resides elsewhere in the engine; this plugin is merely the switch that activates it.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's `PluginLoader` service during server startup. The `JavaPluginInit` context, containing necessary registries and server information, is injected via the constructor.
-   **Scope:** The instance persists for the entire duration of the server session, provided the plugin is not dynamically unloaded.
-   **Destruction:** The object is marked for garbage collection upon server shutdown or when the plugin is unloaded. There are no explicit teardown methods implemented in this class.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It does not define or manage any internal fields. Its function is to modify the external state of the `ClientFeatureRegistry` during a single, idempotent operation.
-   **Thread Safety:** This component is **not thread-safe** and is not designed to be. The plugin lifecycle guarantees that the `setup` method is invoked from a single, main server thread during the initialization sequence. Concurrent invocation of its methods is an unsupported and dangerous operation.

## API Surface
The public contract is defined by the `JavaPlugin` base class. The primary logic is in the protected `setup` method, which is called by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Registers the Mantling feature and associated client tags with the `ClientFeatureRegistry`. This is the core function of the plugin. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is automatically discovered and loaded by the Hytale server's plugin system. The standard "usage" is to ensure the compiled plugin is placed in the server's designated plugins directory. The framework handles the rest.

The conceptual lifecycle, managed by the server, is as follows:
```java
// PSEUDOCODE: Server's PluginLoader
JavaPluginInit initContext = createPluginInitContext();
MantlingPlugin plugin = new MantlingPlugin(initContext);

// The framework invokes the setup method at the correct time
plugin.setup();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new MantlingPlugin()`. The constructor requires a fully-formed `JavaPluginInit` context, which can only be provided by the server's plugin framework. Manual instantiation will result in a non-functional plugin that is disconnected from the server lifecycle.
-   **Manual Invocation:** Do not call the `setup` method directly. This bypasses the server's initialization sequence and can lead to race conditions or an inconsistent server state if the `ClientFeatureRegistry` is not ready to be modified.

## Data Pipeline
This plugin acts as an initial data source for server feature configuration. It does not process incoming data.

> Flow:
> **MantlingPlugin.setup()** -> ClientFeatureRegistry -> Server Feature Set -> Client Handshake Packet -> Client Game Engine

