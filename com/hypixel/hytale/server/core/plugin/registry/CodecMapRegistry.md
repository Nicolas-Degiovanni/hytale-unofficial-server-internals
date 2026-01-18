---
description: Architectural reference for CodecMapRegistry
---

# CodecMapRegistry

**Package:** com.hypixel.hytale.server.core.plugin.registry
**Type:** Transient

## Definition
```java
// Signature
public class CodecMapRegistry<T, C extends Codec<? extends T>> implements IRegistry {
```

## Architecture & Concepts

The CodecMapRegistry serves as a high-level, plugin-facing facade for registering custom data serializers and deserializers (Codecs) with the core engine. It is a fundamental component of the server's plugin architecture, enabling developers to extend the engine's ability to understand and process custom data types, such as new block types, items, or entities.

Architecturally, this class does not implement the registration logic itself. Instead, it acts as a managed handle that delegates all operations to a shared, underlying `StringCodecMapCodec` instance provided by the plugin system. Its primary responsibilities are:

1.  **API Simplification:** To provide a clean, type-safe interface for plugins to register their codecs without exposing the engine's internal codec management machinery.
2.  **Lifecycle Management:** To automatically couple every registration with a corresponding unregistration action. This is critical for enabling robust plugin hot-swapping and clean server shutdowns, preventing memory leaks and state corruption.

The nested `Assets` class is a specialized variant designed specifically for registering `JsonAsset` types, which are a common pattern for defining game content.

## Lifecycle & Ownership

The lifecycle of a CodecMapRegistry instance is intentionally brief and tightly controlled by the plugin loader. Understanding this is critical to avoid resource leaks.

-   **Creation:** An instance is created by the Plugin Management System exclusively during the initialization phase of a specific plugin. It is then passed to the plugin's main entry point. It is **not** a global singleton.
-   **Scope:** The object is transient and is only intended to be used within the scope of a plugin's `onEnable` or equivalent initialization method. Once the plugin has finished its setup, this registry object can and should be garbage collected.
-   **Destruction:** The object itself has no explicit destruction logic; its `shutdown` method is a no-op. The *registrations* it creates, however, are managed by the Plugin Manager. The `unregister` list, which is injected during construction, contains all the cleanup tasks. When a plugin is disabled or reloaded, the Plugin Manager iterates this external list and executes the teardown logic for each registration made through this facade.

## Internal State & Concurrency

-   **State:** The CodecMapRegistry is stateful, but it does not own its state. Its internal fields, `mapCodec` and `unregister`, are references to mutable collections managed by the central Plugin Manager. It acts as a temporary, scoped mutator for this shared state.
-   **Thread Safety:** This class is **not thread-safe**. All registration operations must be performed synchronously on the main server thread during the plugin's initialization phase. The unregistration logic, which is added as a lambda, contains explicit locking on the global `AssetRegistry.ASSET_LOCK`. This indicates that the underlying codec maps are a shared, global resource, and modifications must be carefully synchronized. Asynchronous registration will lead to race conditions and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, aClass, codec) | CodecMapRegistry | O(1) | Registers a codec for a given type. Atomically schedules a corresponding unregistration action for plugin shutdown. |
| register(priority, id, aClass, codec) | CodecMapRegistry | O(1) | Registers a codec with a specific priority, influencing resolution when multiple codecs could apply. |
| shutdown() | void | O(1) | **No-op.** Implemented for the IRegistry interface. Cleanup is managed externally. |
| Assets.register(id, aClass, codec) | CodecMapRegistry.Assets | O(1) | A specialized version for registering `JsonAsset` types using a `BuilderCodec`. |

## Integration Patterns

### Standard Usage

A CodecMapRegistry instance is typically provided by a context object during plugin initialization. The developer uses this instance to register all custom codecs.

```java
// Example from a hypothetical plugin's onEnable() method
public void onEnable(PluginContext context) {
    // Obtain the registry for custom block types
    CodecMapRegistry<CustomBlock, Codec<? extends CustomBlock>> registry = context.getRegistry(RegistryKeys.BLOCKS);

    // Register a new block type with its codec
    registry.register("my_plugin:adamantium_ore", AdamantiumOreBlock.class, new AdamantiumOreCodec());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new CodecMapRegistry()`. The class requires externally managed state (the `mapCodec` and `unregister` list) to function correctly. Attempting to create it manually will result in registrations that are not tracked by the plugin lifecycle, causing resource leaks.
-   **Caching the Registry:** Do not store the CodecMapRegistry instance in a field for later use. It is a transient object meant only for the initialization phase. Attempting to use it after initialization is undefined behavior.
-   **Asynchronous Registration:** Do not call `register` from a separate thread or a scheduled task. All registrations must be completed synchronously on the main thread during plugin loading to prevent race conditions with the underlying shared codec maps.

## Data Pipeline

The CodecMapRegistry is a configuration component that primes a data processing pipeline. It does not process data itself.

**Registration Time (Plugin Load):**
> Flow:
> Plugin `onEnable()` -> **CodecMapRegistry.register()** -> `StringCodecMapCodec` (Internal Engine State)

**Runtime (Data Processing):**
> Flow:
> Network Packet / Disk Asset -> Engine Deserializer -> `StringCodecMapCodec`.lookup("my_plugin:adamantium_ore") -> **(Finds and uses the registered AdamantiumOreCodec)** -> Instantiated `AdamantiumOreBlock` Object

