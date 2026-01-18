---
description: Architectural reference for ObjectiveShopPlugin
---

# ObjectiveShopPlugin

**Package:** com.hypixel.hytale.builtin.adventure.objectiveshop
**Type:** Singleton

## Definition
```java
// Signature
public class ObjectiveShopPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ObjectiveShopPlugin is a bootstrap component responsible for integrating custom objective-related shop functionality into the core server engine. It does not contain runtime game logic itself; instead, its primary function is to perform critical registrations and configurations during the server's startup sequence.

This class acts as a manifest for the objective shop feature, declaring three key pieces of information to the engine:
1.  **Interaction Logic:** It registers the **StartObjectiveInteraction** class, allowing game assets to reference the "StartObjective" action, which the server can then deserialize and execute.
2.  **Requirement Logic:** It registers the **CanStartObjectiveRequirement** class, enabling game assets to define conditions under which the "StartObjective" action is available to a player.
3.  **Asset Dependency:** It explicitly defines a loading dependency, forcing **ShopAsset** to be loaded only after **ObjectiveAsset** is fully loaded. This prevents race conditions where a shop might reference an objective that has not yet been initialized.

In essence, this plugin stitches the objective shop systems into the server's generic asset, interaction, and requirement frameworks.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central **PluginLoader** during the initial bootstrap phase. The engine scans for all classes extending **JavaPlugin** and creates a single instance of each.
-   **Scope:** Persists for the entire server session. The singleton instance is set during the **setup** method call and remains available globally until shutdown.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection during the server shutdown sequence when the **PluginLoader** unloads all active plugins.

## Internal State & Concurrency
-   **State:** The class maintains a single piece of static, mutable state: the **instance** field. This field is written to exactly once during the single-threaded server initialization process. After this point, it is effectively immutable.
-   **Thread Safety:** The **setup** method is invoked by the main server thread and is not thread-safe. However, because it is only ever called within the server's single-threaded startup procedure, this is not a concern. The static **get** method is thread-safe for reads *after* the **setup** method has completed.

    **Warning:** Calling **ObjectiveShopPlugin.get()** from another plugin's constructor or an early initialization block may result in a NullPointerException if this plugin has not yet been loaded. Plugin load order is not guaranteed.

## API Surface
The public API is minimal, as the class's primary purpose is fulfilled via the framework-invoked **setup** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ObjectiveShopPlugin | O(1) | Retrieves the static singleton instance of the plugin. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by other systems. Its functionality is implicitly consumed by the engine systems it configures. For example, a content designer would use the "StartObjective" identifier in a shop asset file, relying on this plugin's registration to make it functional.

A developer would never need to call the **get** method under normal circumstances.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use **new ObjectiveShopPlugin()**. The plugin must be instantiated by the server's **PluginLoader** to ensure its lifecycle is correctly managed and its **setup** method is invoked at the appropriate time.
-   **Early Access:** Do not attempt to retrieve the instance via **get()** during early server initialization phases. This can lead to a NullPointerException due to unpredictable plugin load ordering.

## Data Pipeline
This plugin does not process a continuous stream of data. Instead, it acts as a one-time configuration step that alters two major engine pipelines: Asset Loading and Player Interaction.

**1. Asset Loading Configuration:**
The plugin injects a rule into the **AssetRegistry** to control the initialization order of dependent assets.

> Flow:
> Server Starts -> PluginLoader loads **ObjectiveShopPlugin** -> **setup()** is called -> A rule is injected into **AssetRegistry** -> Later, when assets are loaded -> **ObjectiveAsset** is guaranteed to load before **ShopAsset**.

**2. Interaction & Requirement Registration:**
The plugin provides the necessary codecs to the engine, allowing it to understand custom data types within generic asset files.

> Flow:
> Server Starts -> **ObjectiveShopPlugin.setup()** registers codecs for "StartObjective" and "CanStartObjective" -> Server loads a **ShopAsset** from disk -> The asset's JSON contains "StartObjective" -> The registered codec is used to deserialize this string into a **StartObjectiveInteraction** object.

