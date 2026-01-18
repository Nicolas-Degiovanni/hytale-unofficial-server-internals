---
description: Architectural reference for BlockStateModule
---

# BlockStateModule

**Package:** com.hypixel.hytale.server.core.universe.world.meta
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class BlockStateModule extends JavaPlugin {
```

## Architecture & Concepts

The BlockStateModule is a legacy plugin that serves as a compatibility layer and registration authority for block-related Entity Component System (ECS) components. Its primary function is to dynamically register custom **BlockState** subclasses with the server's core **ChunkStore** ECS registry.

Architecturally, this module acts as a bridge between an older, object-oriented model of special blocks (**BlockState**) and the modern, data-oriented ECS design. When a plugin wishes to define a block with unique behaviors (e.g., a chest with an inventory, a furnace that smelts), it uses this module to inject the corresponding component type and associated processing systems into the world simulation.

**WARNING:** This entire module is deprecated and scheduled for removal. It represents an outdated pattern for extending block functionality. New development should utilize the modern ECS APIs directly, if available, and avoid depending on this module. The systems it registers are explicitly named *Legacy* (e.g., **LegacyTickingBlockStateSystem**) to signify their status.

The module is responsible for:
1.  **Dynamic Component Registration:** Translating a Java class (e.g., **ItemContainerState**) into a formal **ComponentType** within the **ChunkStore** ECS.
2.  **System Injection:** Automatically registering a suite of generic "Legacy" systems that handle the lifecycle, ticking, and network synchronization for any registered **BlockState**.
3.  **Instance Factory:** Providing a mechanism (**createBlockState**) to instantiate **BlockState** objects from serialized data or gameplay events.

## Lifecycle & Ownership

-   **Creation:** A single instance of **BlockStateModule** is instantiated by the server's plugin loader during the server bootstrap sequence. Its creation is dictated by its **PluginManifest**. The static instance is set within the constructor, a common but fragile pattern for service location.
-   **Scope:** The instance is a session-scoped singleton, persisting for the entire lifetime of the server process. It is accessible globally via the static **get** method.
-   **Destruction:** The module is torn down during server shutdown. The **unregisterBlockState** method contains logic to prevent modifications to ECS registries while the server is shutting down, ensuring a cleaner exit.

## Internal State & Concurrency

-   **State:** The module's state is highly mutable. Its core state is the **classToComponentType** map, which caches the mapping between a **BlockState** Java class and its dynamically generated ECS **ComponentType**. This map is populated at runtime as other plugins register their custom block states.
-   **Thread Safety:** The **classToComponentType** map is a **ConcurrentHashMap**, making individual read and write operations on the map itself thread-safe. However, the broader **registerBlockState** operation is not atomic. It performs multiple, non-transactional calls to the underlying ECS registry.

    **WARNING:** Calling **registerBlockState** from multiple threads concurrently or after the server has finished its initial loading phase can lead to severe race conditions and unpredictable behavior within the ECS world. All registration should occur synchronously within a plugin's **setup** method.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | BlockStateModule | O(1) | Retrieves the global singleton instance. |
| registerBlockState(clazz, key, codec, ...) | BlockStateRegistration | O(k) | Registers a BlockState class with the ECS. Injects multiple systems. **This is the primary entry point.** |
| createBlockState(key, chunk, pos, ...) | BlockState | O(k) | Factory method to instantiate a new BlockState from its registered key and initialize it. |
| getComponentType(clazz) | ComponentType | O(1) | Retrieves the cached ComponentType for a given BlockState class. Returns null if not registered or module is disabled. |

## Integration Patterns

### Standard Usage

The intended use is for other plugins to register their custom **BlockState** types during their own initialization phase. This wires the custom block's logic into the server's core simulation loop.

```java
// In another plugin's setup() method:
public class MyCustomPlugin extends JavaPlugin {
    @Override
    protected void setup() {
        BlockStateModule module = BlockStateModule.get();
        if (module != null) {
            module.registerBlockState(
                MyCustomChestState.class,
                "my_mod:custom_chest",
                MyCustomChestState.CODEC
            );
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlockStateModule()`. The server's plugin manager is solely responsible for its lifecycle. Always use the static **get** method.
-   **Late Registration:** Do not call **registerBlockState** after the server has started and worlds are loaded. This can cause inconsistencies where existing chunks do not recognize the new block state type, potentially leading to data loss or crashes.
-   **Reliance on Deprecated Code:** The most significant anti-pattern is using this module for new features. Its existence is for backward compatibility, and it should be avoided in favor of modern, direct ECS patterns.

## Data Pipeline

**BlockStateModule** does not process data in a traditional pipeline. Instead, it configures pipelines for the **ChunkStore** ECS to execute.

> **Registration Flow:**
> Other Plugin's **setup()** -> **BlockStateModule.registerBlockState()** -> **ChunkStore.registerComponent()** -> **ChunkStore.registerSystem()** -> ECS is now aware of the new BlockState type and its logic.

> **Runtime Tick Flow (for a registered TickableBlockState):**
> World Tick -> **ChunkStore** System Scheduler -> **LegacyTickingBlockStateSystem.tick()** -> **TickableBlockState.tick()** -> **CommandBuffer** (State Mutations) -> ECS applies changes.

> **Runtime Network Flow (for a registered SendableBlockState):**
> Player moves near chunk -> **ChunkStore.LoadPacketDataQuerySystem** -> **LegacyLoadPacketBlockStateSystem.fetch()** -> **SendableBlockState.sendTo()** -> List of Packets -> Network Layer.

