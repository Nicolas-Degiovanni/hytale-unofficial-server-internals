---
description: Architectural reference for AssetRegistry
---

# AssetRegistry

**Package:** com.hypixel.hytale.server.core.plugin.registry
**Type:** Transient Handle

## Definition
```java
// Signature
public class AssetRegistry {
```

## Architecture & Concepts
The AssetRegistry class serves as a scoped, temporary handle for registering plugin-specific AssetStore implementations with the global, static asset system. Its primary architectural role is to enforce clean resource management by linking an asset's registration to a plugin's lifecycle.

It acts as a bridge between a plugin's isolated context and the engine's shared `com.hypixel.hytale.assetstore.AssetRegistry`. By capturing unregistration logic at the point of registration, it guarantees that a plugin's assets can be cleanly unloaded from memory when the plugin is disabled or reloaded, preventing resource leaks.

This class embodies the RAII (Resource Acquisition Is Initialization) pattern. The "resource" is the registration within the global asset system, and its lifetime is tied to the plugin that created it, managed via the externally-provided `unregister` list.

### Lifecycle & Ownership
- **Creation:** Instantiated by the plugin loading framework during a plugin's initialization sequence. A reference to a list, owned by the plugin loader, is passed into the constructor. This instance is then typically provided to the plugin's main entry point.
- **Scope:** The AssetRegistry object itself is short-lived and intended for use only within the plugin's startup method. However, the unregistration closures it creates are captured in the `unregister` list and persist for the entire lifetime of the plugin.
- **Destruction:** The object is eligible for garbage collection as soon as the plugin's initialization phase completes. The crucial cleanup logic is not triggered by this object's destruction, but by the plugin framework iterating the `unregister` list during the plugin shutdown sequence.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only field, `unregister`, is a reference to a list managed and owned by an external system (the plugin loader). It does not cache data or maintain its own state.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively by a single thread during the synchronous phase of plugin initialization. The underlying `List` is not concurrent, and parallel calls to `register` would result in a race condition and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(S assetStore) | AssetRegistry | O(1) | Registers the given AssetStore with the global engine registry and schedules it for unregistration upon plugin shutdown. Returns `this` to enable a fluent interface. |
| shutdown() | void | O(1) | **No-op.** This method is empty and has no effect. Cleanup is handled externally. |

## Integration Patterns

### Standard Usage
The intended pattern is for a plugin to receive this object from the framework during its startup, use it to register all necessary asset stores, and then discard the reference.

```java
// Inside a plugin's onEnable(PluginContext context) method
AssetRegistry assetRegistry = context.getAssetRegistry();

assetRegistry
    .register(new CustomBlockStore())
    .register(new CustomItemStore());

// Do not hold a reference to assetRegistry beyond this point.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new AssetRegistry()`. The provided `unregister` list is critical for preventing resource leaks. If you instantiate it directly with a local list, the unregistration logic will be lost when the object is garbage collected.
- **Calling shutdown:** Invoking the `shutdown` method is pointless as it performs no action. Relying on it for cleanup will lead to unexpected behavior.
- **Deferred Registration:** Do not hold onto the AssetRegistry instance to register assets later in the plugin's lifecycle. Registration should only occur during the designated initialization phase.

## Data Pipeline
AssetRegistry is not part of a data processing pipeline. Instead, it manages the control flow for asset lifecycle management.

> **Registration Flow:**
> Plugin Startup -> Plugin Code calls `assetRegistry.register(myStore)` -> **AssetRegistry** adds unregister logic to its list -> **AssetRegistry** calls static `com.hypixel.hytale.assetstore.AssetRegistry.register(myStore)`

> **Unregistration Flow:**
> Plugin Shutdown Trigger -> Plugin Framework iterates the `unregister` list -> Lambda executes `com.hypixel.hytale.assetstore.AssetRegistry.unregister(myStore)`

