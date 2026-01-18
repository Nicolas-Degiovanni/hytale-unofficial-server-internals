---
description: Architectural reference for NPCReputationPlugin
---

# NPCReputationPlugin

**Package:** com.hypixel.hytale.builtin.adventure.npcreputation
**Type:** Plugin EntryPoint

## Definition
```java
// Signature
public class NPCReputationPlugin extends JavaPlugin {
```

## Architecture & Concepts
The NPCReputationPlugin serves as the bootstrap and registration entry point for the entire NPC reputation gameplay feature. It is not a long-lived service that handles game logic directly. Instead, its primary architectural role is to bridge the server's plugin loading mechanism with the core Entity-Component-System (ECS) engine.

Upon initialization, this plugin registers two critical systems with the server's EntityStoreRegistry:
1.  **ReputationAttitudeSystem:** A system responsible for translating raw reputation values into observable in-game behaviors or attitudes for NPCs.
2.  **NPCReputationHolderSystem:** A system that manages the association between an NPC entity and its corresponding reputation data, likely ensuring data consistency and handling updates.

By registering these systems, the plugin declaratively injects the reputation logic into the main server game loop. The systems themselves contain the actual logic and are executed by the ECS engine on every relevant tick.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's internal plugin loader during the server startup sequence or when a new plugin is loaded at runtime. The `JavaPluginInit` context, containing references to core registries, is injected via the constructor.
-   **Scope:** Session-scoped. The plugin object persists for the entire duration of the server's runtime or until it is explicitly unloaded by an administrator. The systems it registers share this lifecycle.
-   **Destruction:** Decommissioned during server shutdown or a plugin unload event. The base `JavaPlugin` class is expected to handle the graceful deregistration of its associated systems from the ECS engine to prevent memory leaks or stale logic.

## Internal State & Concurrency
-   **State:** This class is stateless. It does not define or manage any fields of its own. Its purpose is fulfilled entirely within the `setup` method, acting as a transient configuration object during the initialization phase.
-   **Thread Safety:** This class is not thread-safe and is not designed for concurrent access. The plugin lifecycle methods, particularly `setup`, are guaranteed by the framework to be called from a single main server thread. Invoking its methods from other threads is an unsupported operation and will lead to race conditions in the EntityStoreRegistry.

## API Surface
The public contract is defined by the `JavaPlugin` base class. The only method of significance is the `setup` override.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | void | O(1) | Registers the required NPC reputation systems with the core ECS engine. This method is called automatically by the plugin framework and should not be invoked manually. |

## Integration Patterns

### Standard Usage
This class is not intended to be used programmatically by other developers. Its integration is declarative. A server operator enables the NPC reputation feature by including this plugin in the server's active plugin list. The server's plugin framework handles the entire lifecycle.

There is no standard code-based usage pattern.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new NPCReputationPlugin()`. The plugin loader is the sole owner of this class's lifecycle and is responsible for injecting the required `JavaPluginInit` context. Manual instantiation will result in a non-functional plugin.
-   **Manual Invocation:** Do not call the `setup` method directly. This bypasses the managed plugin lifecycle, which can lead to duplicate system registrations, initialization errors, or unpredictable behavior if called at the wrong time.

## Data Pipeline
This plugin does not directly process data. Instead, it *configures* the data processing pipeline by injecting systems into the ECS engine. The flow it enables is as follows:

> Flow:
> Server Plugin Loader -> **NPCReputationPlugin.setup()** -> EntityStoreRegistry -> [Game Loop Tick] -> ReputationAttitudeSystem & NPCReputationHolderSystem -> NPC Component Data Update

