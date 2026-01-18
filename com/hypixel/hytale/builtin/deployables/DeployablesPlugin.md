---
description: Architectural reference for DeployablesPlugin
---

# DeployablesPlugin

**Package:** com.hypixel.hytale.builtin.deployables
**Type:** Singleton

## Definition
```java
// Signature
public class DeployablesPlugin extends JavaPlugin {
```

## Architecture & Concepts
The DeployablesPlugin serves as the central bootstrap and registration authority for the entire Deployables feature set within the Hytale server. It is not a runtime service that performs continuous work; rather, its primary function is to be executed once during server initialization to integrate all necessary assets, components, and systems into the core engine.

This class follows the **Plugin** architectural pattern, encapsulating a discrete feature and injecting it into the main application via the `setup` lifecycle method. It acts as a manifest, declaring the existence and configuration of several key architectural components:

*   **Asset Types:** It registers the `DeployableSpawner` asset type with the global AssetRegistry. This informs the asset pipeline how to load, parse, and manage configuration files for deployable items, and critically, defines its loading order relative to other assets like models and sounds.
*   **ECS Components:** It registers all data components related to deployables, such as DeployableComponent and DeployableOwnerComponent, with the server's Entity-Component-System (ECS) framework. This allows entities in the world to possess state related to deployable behavior.
*   **ECS Systems:** It registers the logic-bearing `DeployablesSystem` variants. These systems are responsible for processing entities that have deployable components during the main game loop tick.
*   **Configuration Codecs:** It extends the engine's data serialization capabilities by registering specific codecs for different deployable configurations (e.g., Trap, Turret) and custom interactions. This enables the engine to correctly parse these types from game data files.

In essence, the DeployablesPlugin is the glue that binds the deployables feature to the server's foundational ECS, asset, and interaction frameworks.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the Hytale server's internal PluginLoader during the server bootstrap sequence. Manual instantiation is strictly forbidden.
- **Scope:** Application-scoped singleton. A single instance is created and persists for the entire lifetime of the server process.
- **Destruction:** The instance is destroyed when the server shuts down and the PluginLoader unloads all active plugins. There is no explicit public teardown method; cleanup is managed by the server and the Java Virtual Machine.

## Internal State & Concurrency
- **State:** The internal state is effectively **immutable after initialization**. The fields, such as the static instance and the various ComponentType handles, are populated once within the `setup` method. Subsequent operations are read-only. The plugin does not cache runtime data; it holds static references and registration handles.

- **Thread Safety:** The class is **conditionally thread-safe**.
    - The `setup` method is non-reentrant and **must** be executed only once by the main server thread during its initialization phase.
    - After initialization is complete, all public methods (e.g., `get`, `getDeployableComponentType`) are thread-safe as they return immutable or thread-safe handles. Accessing the plugin from multiple threads for read-only operations post-setup is safe and supported.

    **WARNING:** Any attempt to access the singleton instance via `get()` before the plugin loader has completed the `setup` call will result in a NullPointerException.

## API Surface
The public API is minimal and focused on providing access to the registration handles created during setup.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static DeployablesPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| getDeployableComponentType() | ComponentType | O(1) | Returns the engine handle for the DeployableComponent type. |
| getDeployableOwnerComponentType() | ComponentType | O(1) | Returns the engine handle for the DeployableOwnerComponent type. |
| getDeployableProjectileShooterComponentType() | ComponentType | O(1) | Returns the engine handle for the DeployableProjectileShooterComponent type. |
| getDeployableProjectileComponentType() | ComponentType | O(1) | Returns the engine handle for the DeployableProjectileComponent type. |

## Integration Patterns

### Standard Usage
The intended use is for other systems or plugins to retrieve the singleton instance and then use its getter methods to acquire the necessary ComponentType handles for interacting with the ECS framework.

```java
// Example from another game system that needs to add a deployable component to an entity.
DeployablesPlugin plugin = DeployablesPlugin.get();

// Retrieve the type handle once and cache it if performance is critical.
ComponentType<EntityStore, DeployableComponent> deployableType = plugin.getDeployableComponentType();

// Use the handle to add a new component to a world entity.
// Assumes 'myEntity' is a valid entity object.
myEntity.addComponent(deployableType);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DeployablesPlugin()`. The plugin's lifecycle is exclusively managed by the server's plugin loader. Doing so will result in an uninitialized and non-functional object.
- **Access Before Initialization:** Do not call `DeployablesPlugin.get()` from the constructor of another plugin that may load before this one. This creates a race condition where the static `instance` field is null, leading to a fatal NullPointerException.
- **State Modification:** Do not attempt to use reflection or other means to modify the state of the plugin after the `setup` method has completed. The internal handles are assumed to be constant for the server's lifetime.

## Data Pipeline
This class is not a direct participant in a runtime data pipeline. Instead, its `setup` method is a **meta-process that configures and injects other components into multiple engine pipelines**.

> **Asset Loading Pipeline Configuration:**
> `deployable.json` on disk -> Server Asset Loader -> **AssetRegistry (using DeployableSpawner.CODEC registered by DeployablesPlugin)** -> In-memory `DeployableSpawner` object

> **ECS Game Loop Integration:**
> Game Tick -> **DeployablesSystem (registered by DeployablesPlugin)** -> Queries for entities with `DeployableComponent` -> Updates entity state (e.g., turret aiming, trap triggering)

