---
description: Architectural reference for PrefabSpawnerModule
---

# PrefabSpawnerModule

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner
**Type:** Managed Singleton (Plugin)

## Definition
```java
// Signature
public class PrefabSpawnerModule extends JavaPlugin {
```

## Architecture & Concepts
The PrefabSpawnerModule serves as the primary bootstrap and integration point for the server-side prefab spawning feature. It is not a service that contains business logic, but rather a declarative **module** responsible for registering the feature's components with the core server engine during its startup sequence.

Its fundamental role is to act as a bridge between the Prefab Spawner feature and two critical engine systems:
1.  **The World State System:** By registering the PrefabSpawnerState, it enables the game world to recognize, store, and serialize a custom block entity responsible for spawning prefabs.
2.  **The Command System:** By registering the PrefabSpawnerCommand, it exposes administrative control over the feature via the server console or in-game chat.

The static MANIFEST field is a critical component of the server's module loading system. It explicitly declares a dependency on the BlockStateModule, guaranteeing that the core block system is initialized *before* this module attempts to register its custom block state. This enforces a safe and predictable server initialization order.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's central PluginLoader during the server bootstrap phase. The framework supplies the JavaPluginInit context object, which contains handles to the necessary engine registries.
-   **Scope:** Session-scoped. The PrefabSpawnerModule instance persists for the entire lifetime of the running server.
-   **Destruction:** The instance is discarded and garbage collected during server shutdown when the PluginLoader unloads all active plugins. No explicit cleanup logic is required by this module.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless**. After the one-time execution of its setup method, the instance holds no mutable state. All state related to the feature (e.g., the list of registered commands) is owned and managed by the respective core engine registries.
-   **Thread Safety:** The setup method is invoked by the server's main thread during the single-threaded initialization phase, making it inherently safe. The components it registers (PrefabSpawnerState and PrefabSpawnerCommand) are expected to adhere to the concurrency contracts of the systems they integrate with. This module itself presents no concurrency concerns.

## API Surface
The public API is designed for framework interaction only and should not be invoked by other services or plugins.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Registers the module's components. Invoked by the JavaPlugin framework. |

**WARNING:** The public constructor is an implementation detail of the plugin system. Direct invocation will bypass the server's lifecycle management and lead to a highly unstable state.

## Integration Patterns

### Standard Usage
This module is not used directly. It is enabled by being included in a server's configuration, and the server runtime handles its lifecycle automatically. A user interacts with the *features* this module provides, typically via the registered command.

```java
// An administrator or player would use the registered command in-game
// This example is conceptual and represents the result of the module's work.

/prefab spawn my_cool_fortress
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PrefabSpawnerModule()`. The plugin system is the sole owner and creator of module instances. Doing so will result in an uninitialized and non-functional object.
-   **Manual Invocation:** Do not call the `setup` method manually. This will either throw a NullPointerException if the module is not properly initialized or cause duplicate registration errors if called after the server has already loaded it.

## Data Pipeline
This module is an initializer, not a data processor. It does not participate in a data pipeline itself but rather *enables* pipelines for the components it registers. The most prominent pipeline is the command processing flow.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Parser -> **PrefabSpawnerCommand** (Registered by this module) -> World Manipulation API -> Prefab Instantiation Logic

