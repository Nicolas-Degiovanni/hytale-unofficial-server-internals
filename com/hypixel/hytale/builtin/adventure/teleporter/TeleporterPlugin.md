---
description: Architectural reference for TeleporterPlugin
---

# TeleporterPlugin

**Package:** com.hypixel.hytale.builtin.adventure.teleporter
**Type:** Singleton

## Definition
```java
// Signature
public class TeleporterPlugin extends JavaPlugin {
```

## Architecture & Concepts
The TeleporterPlugin serves as the central initialization and registration authority for the entire teleporter feature set within the Hytale server. As a subclass of JavaPlugin, it is automatically discovered and loaded by the server's plugin manager during the bootstrap sequence.

Its primary architectural role is not to execute runtime game logic, but to configure the server's core systems to understand and manage teleporter-related behaviors. It acts as a manifest, declaring all the necessary components, systems, and data codecs that constitute the teleporter feature.

Specifically, the plugin integrates with the server's Entity Component System (ECS) and other core modules by:
1.  **Registering the Teleporter Component:** It defines the Teleporter data component, allowing entities to possess teleporter properties that are managed by the ECS framework.
2.  **Registering Systems:** It injects the core logic of the teleporter feature by registering several systems. These systems subscribe to ECS events (e.g., component addition, removal, entity destruction) and execute game logic in response, such as creating or deleting associated warp points.
3.  **Registering Codecs:** It registers custom codecs for player interactions and UI pages. This enables the server to serialize and deserialize teleporter-specific interaction data and link in-world teleporter objects to their corresponding settings UI.

This class is the bridge between the generic, engine-level ECS framework and the high-level, domain-specific logic of the teleporter game feature. It also establishes a critical dependency on the global TeleportPlugin to manage the persistence of warp points.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the Hytale Server's plugin loader during server startup. The constructor is invoked via reflection; direct instantiation is prohibited. The static `instance` field is set within the constructor, enforcing the singleton pattern.
-   **Scope:** Session-scoped. The TeleporterPlugin instance persists for the entire lifetime of the server process.
-   **Destruction:** The object is destroyed when the server shuts down. The Java Virtual Machine performs cleanup, and there is no explicit public-facing destruction or teardown method.

## Internal State & Concurrency
-   **State:** The class maintains a minimal, effectively immutable state after initialization. The `teleporterComponentType` field is written to exactly once during the `setup` phase and is read-only thereafter. The static `instance` field is also set once. The plugin does not cache runtime game data; it delegates state management to the ECS and the TeleportPlugin.
-   **Thread Safety:** The `setup` method is guaranteed by the plugin framework to be called from a single thread during the server's synchronous initialization phase. All subsequent access to its internal state is read-only. Therefore, the class itself is thread-safe.

    **WARNING:** While the plugin is thread-safe, the systems it registers (e.g., TeleporterOwnedWarpRefSystem) are executed by the ECS scheduler. These systems must be designed to handle concurrent execution if the underlying ECS engine is multi-threaded.

## API Surface
The public API is minimal, designed to provide essential, read-only information to other parts of the engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static TeleporterPlugin | O(1) | Retrieves the global singleton instance of the plugin. Throws NullPointerException if called before the plugin is loaded. |
| getTeleporterComponentType() | ComponentType | O(1) | Returns the registered ComponentType for the Teleporter component. Essential for querying the ECS for teleporter entities. |

## Integration Patterns

### Standard Usage
Developers should never interact with the plugin's lifecycle methods. The primary and intended use case is to retrieve the singleton instance to access the registered ComponentType, which is then used to build ECS queries.

```java
// In another system or plugin, retrieve the component type to find all teleporters.
TeleporterPlugin plugin = TeleporterPlugin.get();
ComponentType<ChunkStore, Teleporter> teleporterType = plugin.getTeleporterComponentType();

// Use the type to construct a query for the ECS.
Query<ChunkStore> allTeleportersQuery = Query.is(teleporterType);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new TeleporterPlugin()`. The server's plugin loader is the sole owner of its lifecycle. Doing so will break the singleton pattern and fail to register the feature correctly.
-   **Manual Lifecycle Management:** Do not call the `setup` method manually. This will cause duplicate registrations, leading to registry corruption and unpredictable server behavior.
-   **Premature Access:** Do not call `TeleporterPlugin.get()` from a plugin that does not declare a dependency on this one. This can create a race condition where the `instance` is accessed before it has been initialized by the plugin loader, resulting in a NullPointerException.

## Data Pipeline
The TeleporterPlugin is an initializer, not a direct participant in a continuous data pipeline. However, it is responsible for injecting systems that *do* form a pipeline. The following illustrates the data flow when a teleporter entity is destroyed.

> Flow: Teleporter Destruction
>
> Player Action (e.g., breaks block) -> Server Engine marks entity for removal -> ECS Scheduler dispatches `onEntityRemove` event -> **TeleporterOwnedWarpRefSystem** (registered by this plugin) receives event -> System queries the entity for its **Teleporter** component -> System extracts the `ownedWarp` name -> System calls `TeleportPlugin.get().getWarps().remove()` -> Global warp map is modified -> `TeleportPlugin` persists the change to disk.

