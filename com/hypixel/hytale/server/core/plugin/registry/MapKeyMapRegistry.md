---
description: Architectural reference for MapKeyMap-based registration management.
---

# MapKeyMapRegistry

**Package:** com.hypixel.hytale.server.core.plugin.registry
**Type:** Transient

## Definition
```java
// Signature
public class MapKeyMapRegistry<V> implements IRegistry {
```

## Architecture & Concepts

The MapKeyMapRegistry serves as a high-level facade for registering serializable types within the server's codec system. It is a critical component of the plugin and content loading infrastructure, providing a structured and lifecycle-aware mechanism for associating a Java class with a unique string identifier and its corresponding serialization logic (a Codec).

Architecturally, this class does not maintain the registration state itself. Instead, it acts as a transient, fluent configurator for a shared, underlying **MapKeyMapCodec** instance. Its primary responsibility is to simplify the registration process for developers while simultaneously enrolling the registration into the server's broader lifecycle management system. This is achieved by injecting a shared list of unregistration callbacks, ensuring that any type registered through this class can be cleanly removed during events like a plugin unload or a server hot-reload.

This design decouples the act of registration from the management of the registration's lifecycle, promoting a robust and modular plugin environment.

## Lifecycle & Ownership

-   **Creation:** An instance of MapKeyMapRegistry is created by a parent registry or a plugin loader during the server's initialization phase. It is constructed with two critical, externally-owned dependencies: the target **MapKeyMapCodec** to be modified, and a shared **List** of unregistration callbacks. It is never intended to be instantiated directly by end-users.
-   **Scope:** The object's lifetime is intentionally brief. It typically exists only for the duration of a specific module's or plugin's registration block.
-   **Destruction:** The MapKeyMapRegistry object itself is eligible for garbage collection once the registration block completes. However, its effects are persistent: the registration data lives on in the injected MapKeyMapCodec, and the cleanup callback it creates is stored in the shared unregister list until a shutdown or unload event triggers its execution.

## Internal State & Concurrency

-   **State:** This class is effectively stateless. Its fields are final references to externally managed, mutable collections. It performs no internal caching or state modification beyond delegating to these injected dependencies.
-   **Thread Safety:** The class is **not thread-safe** for concurrent invocations on a single instance. Registrations should be performed serially during a controlled initialization phase.

    However, the unregistration logic produced by this class is designed for concurrent environments. The generated callback acquires a global write lock on **AssetRegistry.ASSET_LOCK** before attempting to modify the underlying codec. This is a critical safety mechanism that prevents race conditions and data corruption during complex operations like a parallelized plugin unload or a live asset reload.

    **Warning:** Failure to respect the single-threaded registration model can lead to unpredictable behavior in the shared callback list.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(Class, String, Codec) | MapKeyMapRegistry | O(1) | Registers a class, its string ID, and its codec with the underlying system. Enqueues a corresponding unregistration task. Returns itself for fluent chaining. |
| shutdown() | void | O(1) | No-op. This method fulfills the IRegistry interface contract, but the actual shutdown logic is handled externally via the unregistration callbacks. |

## Integration Patterns

### Standard Usage

The MapKeyMapRegistry is intended to be received from a context or event during plugin loading. The developer uses the provided instance to register all necessary types.

```java
// Example from a hypothetical plugin initialization method
void onRegisterTypes(RegistryContext ctx) {
    MapKeyMapRegistry<Block> blockRegistry = ctx.getBlockRegistry();

    blockRegistry
        .register(StoneBlock.class, "myplugin:stone", new StoneBlockCodec())
        .register(DirtBlock.class, "myplugin:dirt", new DirtBlockCodec());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new MapKeyMapRegistry()`. Doing so will fail to connect it to the server's central codec and lifecycle management systems, resulting in registrations that are not recognized and cannot be cleaned up.
-   **Storing References:** Do not cache or store a reference to this object beyond the scope of the initialization block. It is a transient utility, and its underlying dependencies may become invalid later in the server lifecycle.
-   **Manual Unregistration:** Do not attempt to manually unregister types from the underlying codec. This bypasses the managed lifecycle and can break the server's hot-reload and plugin unload functionality.

## Data Pipeline

This class functions as a configuration component for the server's data serialization pipeline. It does not process data directly but rather sets up the rules for how data will be processed later.

> **Configuration Flow:**
> Plugin Initialization -> **MapKeyMapRegistry.register()** -> Injects (Class -> ID, Codec) mapping into -> **MapKeyMapCodec**
>
> **Later Data Flow:**
> Network Packet -> Deserializer uses **MapKeyMapCodec** to find Codec by ID -> Codec instantiates Class object

