---
description: Architectural reference for ShopReputationPlugin
---

# ShopReputationPlugin

**Package:** com.hypixel.hytale.builtin.adventure.shopreputation
**Type:** Plugin Component

## Definition
```java
// Signature
public class ShopReputationPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ShopReputationPlugin serves a single, critical purpose: it establishes a load-time dependency between the Shop system and the Reputation system. It is a pure bootstrap configuration component, containing no runtime logic.

Its primary function is to interface with the global AssetRegistry during the server's initialization phase. By calling `injectLoadsAfter`, it instructs the asset loading pipeline that all ShopAsset definitions must be processed only *after* ReputationGroup and ReputationRank assets have been fully loaded and registered.

This explicit ordering is essential for data integrity. It allows ShopAsset definitions to safely reference reputation data, such as requiring a specific reputation rank to unlock an item. Without this plugin, the asset loader might attempt to process a ShopAsset before its dependent ReputationAsset exists, leading to unresolved references and server startup failures. This class acts as the declarative glue between two otherwise decoupled game systems.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's core PluginLoader during the initial bootstrap sequence. The framework supplies the JavaPluginInit context required by the constructor. Direct instantiation by user code is not supported and will result in system instability.
-   **Scope:** The object instance persists for the entire server session. However, its functional relevance is confined to the startup phase. After the `setup` method completes, the plugin instance becomes dormant, holding no state and performing no further actions.
-   **Destruction:** The object is marked for garbage collection during the server shutdown process when the PluginLoader unloads all active plugins.

## Internal State & Concurrency
-   **State:** This class is entirely stateless. It does not contain any member fields and does not cache any data. Its methods operate exclusively on parameters and global static systems like the AssetRegistry.
-   **Thread Safety:** The plugin framework guarantees that the `setup` method is invoked from a single, controlled thread during server initialization. Therefore, the class itself does not require internal synchronization mechanisms. All interactions with the AssetRegistry within this context are considered safe.

## API Surface
The public contract is minimal and intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | void | O(1) | Overrides JavaPlugin. Configures the AssetRegistry to enforce a load order dependency. This is the sole entry point for the plugin's logic. |

## Integration Patterns

### Standard Usage
This plugin is loaded automatically by the server and requires no direct interaction. Developers creating similar dependency-enforcing plugins should follow this pattern, extending JavaPlugin and placing configuration logic within the `setup` method.

```java
// This class is not used directly.
// The server's PluginLoader discovers and loads it at startup.
// The following demonstrates the conceptual lifecycle:

// 1. Server starts.
// 2. PluginLoader scans for all classes extending JavaPlugin.
// 3. PluginLoader instantiates ShopReputationPlugin.
// 4. PluginLoader calls the setup() method.
// 5. AssetRegistry is now configured with the new dependency rule.
// 6. Server proceeds to load all game assets.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ShopReputationPlugin()`. The plugin lifecycle is managed exclusively by the server's core. Manual creation will bypass necessary initialization and result in a non-functional object.
-   **Manual Invocation:** Do not invoke the `setup` method directly. This can corrupt the state of the AssetRegistry if called outside of the designated server bootstrap phase, leading to race conditions or unpredictable asset loading behavior.

## Data Pipeline
This component does not process a stream of data. Instead, it configures the control flow of the server's asset loading pipeline.

> **Control Flow:**
> Server Bootstrap → Plugin Loader Discovers Plugin → **ShopReputationPlugin.setup()** → AssetRegistry Dependency Graph is Modified → Asset Loader Begins Processing → Reputation Assets Loaded → Shop Assets Loaded

