---
description: Architectural reference for ModelPlugin
---

# ModelPlugin

**Package:** com.hypixel.hytale.builtin.model
**Type:** Plugin

## Definition
```java
// Signature
public class ModelPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ModelPlugin is a server-side system responsible for the live-reloading and dynamic updating of entity models. It acts as a critical bridge between the Hytale Asset Store and the Entity Component System (ECS).

Its primary function is to listen for asset hot-reloading events, specifically for ModelAssets. When a model is updated on the server's filesystem, this plugin ensures that all existing entities in every world using that model are seamlessly updated to the new version without requiring a server restart or manual intervention. This enables real-time iteration for artists and designers.

Architecturally, this class embodies an event-driven, reactive pattern. It registers itself as a listener for specific system events and executes highly parallelized logic across the entire server state in response.

### Lifecycle & Ownership
- **Creation:** A single instance of ModelPlugin is created by the server's PluginLoader during the server bootstrap phase. The constructor receives a JavaPluginInit context object, which provides access to core server registries.
- **Scope:** The instance persists for the entire duration of the server session. It is a long-lived, foundational service.
- **Destruction:** The plugin is destroyed during the server shutdown sequence. The JavaPlugin base class handles the unregistration of its listeners and commands.

## Internal State & Concurrency
- **State:** The ModelPlugin is effectively stateless. It does not maintain any internal caches or mutable data structures related to models or entities. Its operations are based entirely on the event payload it receives and the current state of the game worlds.

- **Thread Safety:** This component is designed for high-concurrency environments. The core logic in updateModelAssets is thread-safe due to its adherence to Hytale's ECS concurrency model:
    1. The operation is submitted to each World's dedicated execution queue via world.execute.
    2. Within the world, the entity store is iterated in parallel using forEachEntityParallel.
    3. **Crucially, direct modification of components is forbidden.** Instead, changes are recorded as commands into a thread-local CommandBuffer.
    4. After the parallel iteration completes, the engine merges all command buffers and applies the changes sequentially in a safe, deterministic manner. This command-based pattern prevents race conditions and ensures data integrity.

## API Surface
The public contract is established through the JavaPlugin lifecycle, primarily the setup method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Registers the ModelCommand and the event listener for LoadedAssetsEvent. This is the primary entry point, called once by the plugin system after instantiation. |

## Integration Patterns

### Standard Usage
This plugin is not intended for direct invocation by other plugins or game code. It is a core system that is automatically loaded by the server. Its functionality is triggered implicitly by the Hytale Asset Store.

The primary interaction pattern is event-driven. The system operates as follows:
1. The server's Asset Store detects a change to a model asset file.
2. The Asset Store reloads the asset and fires a LoadedAssetsEvent.
3. The EventRegistry, configured by this plugin's setup method, dispatches the event to the updateModelAssets method.
4. The plugin's logic executes, updating all relevant entities.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ModelPlugin()`. The plugin's lifecycle is exclusively managed by the server's PluginLoader. Manual instantiation will result in a non-functional object that is not registered with any server systems.
- **Manual Invocation:** Do not attempt to call the updateModelAssets or checkForModelUpdate methods directly. These methods are designed to be called by the event system and the internal parallel iterator, respectively. Bypassing the event system can lead to severe concurrency issues and world corruption.

## Data Pipeline
The plugin processes data flowing from the asset system to the entity component system.

> Flow:
> Filesystem Change -> Asset Store -> **LoadedAssetsEvent** -> Event Registry -> **ModelPlugin.updateModelAssets** -> World Executor -> Parallel Entity Iteration -> **CommandBuffer** -> ECS Update -> Network Replication

