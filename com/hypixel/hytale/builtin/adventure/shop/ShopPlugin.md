---
description: Architectural reference for ShopPlugin
---

# ShopPlugin

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Singleton

## Definition
```java
// Signature
public class ShopPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ShopPlugin serves as the central bootstrap and registration authority for the entire server-side shop system. As a subclass of JavaPlugin, it integrates directly into the server's core lifecycle, ensuring that all necessary shop-related components are initialized before the game world becomes active.

Its primary architectural role is not to manage active shop transactions, but to act as a declarative manifest. During the server's `setup` phase, it registers custom asset types, data serializers (codecs), and UI interaction handlers with the engine's core registries. This allows the rest of the Hytale engine—such as the asset loader, networking layer, and UI system—to understand and process shop-specific data without being tightly coupled to the shop logic itself.

This class is the foundational pillar upon which all shop functionality is built. Without its successful initialization, the server would be incapable of loading shop definitions from files, serializing shop data for network transmission, or rendering shop user interfaces for players.

### Lifecycle & Ownership
-   **Creation:** The ShopPlugin is instantiated once by the server's Plugin Loader during the initial server startup sequence. The framework provides a JavaPluginInit context object to its constructor, which contains the necessary handles to core engine registries.

-   **Scope:** The plugin instance is a server-scoped singleton that persists for the entire lifetime of the server process. A static reference, `instance`, is set within the `setup` method to provide global access.

-   **Destruction:** The `shutdown` method is invoked by the server during its graceful shutdown procedure. This triggers the finalization logic, such as saving the BarterShopState. The object is eligible for garbage collection only when the server process terminates.

## Internal State & Concurrency
-   **State:** This class is largely stateless. Its primary mutable field is the static `instance` reference, which is written to exactly once during the `setup` phase. It does not cache shop data or player state; instead, it initializes and delegates state management to dedicated components like BarterShopState.

-   **Thread Safety:** The plugin's lifecycle methods (`setup`, `start`, `shutdown`) are guaranteed by the Hytale server framework to be called sequentially from the main server thread. Therefore, no internal locking is required within these methods. The static `get()` method provides safe read-only access to the fully initialized singleton instance after the `setup` phase is complete.

    **Warning:** Calling any methods on this class from asynchronous tasks or other threads is not supported and may lead to race conditions with the server's main lifecycle loop.

## API Surface
The primary public contract is the static accessor for the singleton instance. Lifecycle methods are protected and intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static ShopPlugin | O(1) | Returns the global singleton instance of the plugin. Throws NullPointerException if called before the plugin's `setup` method has completed. |

## Integration Patterns

### Standard Usage
Direct interaction with the ShopPlugin instance is uncommon. Its main purpose is to perform initial setup. Other systems interact with the *results* of its setup, not the plugin itself. However, if a system needed access to a plugin-scoped resource, it would use the static accessor.

```java
// Example of another system accessing the plugin (hypothetical)
ShopPlugin plugin = ShopPlugin.get();
// Use the plugin instance to access plugin-specific managers or data
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ShopPlugin()`. The server's Plugin Loader is solely responsible for its creation and lifecycle management. Manual instantiation will result in a non-functional, rogue instance that is not registered with the engine.
-   **Manual Lifecycle Invocation:** Do not call `setup()`, `start()`, or `shutdown()` directly. These methods are part of a strict contract with the server framework and calling them out of sequence will corrupt server state.
-   **Early Access:** Do not call `ShopPlugin.get()` from the constructor or `setup` method of another plugin that may load before this one. This can result in a NullPointerException as the `instance` field will not have been initialized yet.

## Data Pipeline
ShopPlugin is not a component in a runtime data pipeline; it is the *architect* of several configuration and data-definition pipelines that are executed during server startup.

> **Configuration Flow:**
> Server Bootstrap → Plugin Loader Instantiates **ShopPlugin** → `setup()` Method Invoked
>
> 1.  **Asset Registration:** `ShopAsset` and `BarterShopAsset` types are registered with the global `AssetRegistry`. The engine now knows how to find, load, and parse `.json` files in the `Shops/` and `BarterShops/` directories.
> 2.  **Codec Registration:** Custom codecs for `ShopElement`, `GiveItemInteraction`, and `ShopPageSupplier` are registered with the `CodecRegistry`. This enables the networking and persistence layers to serialize and deserialize these specific data structures.
> 3.  **State Initialization:** The `start()` method is invoked, which in turn calls `BarterShopState.initialize()`, preparing the barter system's persistent state for runtime operations.

