---
description: Architectural reference for EntityModule
---

# EntityModule

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Singleton

## Definition
```java
// Signature
public class EntityModule extends JavaPlugin {
```

## Architecture & Concepts

The EntityModule is the foundational pillar of the server's Entity Component System (ECS) architecture. It acts as a centralized registrar and bootstrap manager for all entity-related logic, including components, systems, and legacy entity class mappings. As a core JavaPlugin, it is loaded once at server startup and persists for the entire application lifecycle.

Its primary responsibilities are:

1.  **ECS Bootstrap:** The `setup` method is the single largest and most critical initialization point in the entity system. It programmatically registers hundreds of ComponentTypes, ResourceTypes, and Systems with the global ComponentRegistryProxy. These registrations define the data (Components) and logic (Systems) that govern every entity in every World. It configures systems for physics, networking, player management, AI, spatial partitioning, and more.

2.  **Legacy Bridge:** The module provides a critical bridge between the older, object-oriented `Entity` class hierarchy and the modern data-oriented ECS design. Through the `registerEntity` method, it maps traditional Java classes (e.g., Player, BlockEntity) to a corresponding ComponentType in the ECS. Inner systems like `LegacyEntityHolderSystem` and `LegacyEntityRefSystem` are automatically registered to manage the lifecycle of these legacy objects, synchronizing their state with their ECS representation.

3.  **Service Locator for Core Types:** The module serves as a static service locator for obtaining handles to essential, engine-defined ComponentTypes and ResourceTypes. Methods like `getPlayerComponentType` or `getTransformComponentType` provide a stable, centralized API for other game systems and plugins to interact with the ECS without needing to manage component registration themselves.

4.  **Asset Integration:** It registers and manages entity-related assets, such as `MovementConfig` and `HitboxCollisionConfig`. It contains event listeners that react to the loading of these assets, dispatching configuration updates to the relevant entities and systems across all running Worlds. This decouples entity behavior from hardcoded values, allowing for data-driven configuration.

**WARNING:** This module is the source of truth for the server's entity architecture. Any failure during its `setup` phase is considered a fatal server error.

### Lifecycle & Ownership

-   **Creation:** Instantiated once by the server's core plugin loader during application bootstrap. The static `instance` field is set in the constructor, enforcing its singleton nature.
-   **Scope:** Global singleton. The instance persists for the entire server session. It is never reloaded or replaced.
-   **Destruction:** The module is torn down only during a full server shutdown. Its state is not designed to be reset or cleared during runtime.

## Internal State & Concurrency

-   **State:** The EntityModule is highly stateful but its state is almost exclusively populated during the initial, single-threaded `setup` phase. It maintains multiple `ConcurrentHashMap` instances to store mappings between entity class names, class objects, constructors, and their corresponding ECS ComponentTypes. It also holds direct references to every core ComponentType and ResourceType it registers. After startup, this state is effectively immutable.

-   **Thread Safety:** The module is designed for safe concurrent access after initialization.
    -   Registries like `idMap` and `classIdMap` use `ConcurrentHashMap` to permit thread-safe lookups, which is critical for plugins that may register entities from different threads.
    -   Event handlers, such as `onMovementConfigLoadedAssetsEvent`, receive events and immediately delegate the work to the appropriate World's execution thread via `world.execute()`. This pattern ensures that all modifications to ECS data occur within the correct, thread-safe context of a specific World instance.

    **WARNING:** While lookups are thread-safe, calling registration methods like `registerEntity` after the server has finished starting is not a standard pattern and may lead to unpredictable behavior. All registrations should occur during the plugin setup phase.

## API Surface

The public API is primarily for registration and service location. Getters for all 50+ ComponentTypes are omitted for brevity but follow the pattern `get{ComponentName}ComponentType`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | EntityModule | O(1) | Provides static access to the singleton instance. |
| registerEntity(id, class, constructor, codec) | EntityRegistration | O(1) | Registers a legacy Entity class with the ECS. This is the primary extension point for custom entities. |
| getIdentifier(class) | String | O(1) | Retrieves the registered string ID for a given Entity class. |
| getClass(name) | Class | O(1) | Retrieves the registered Entity class for a given string ID. |
| getComponentType(class) | ComponentType | O(1) | Retrieves the master ComponentType for a registered legacy Entity class. |
| getPlayerComponentType() | ComponentType | O(1) | Provides access to the core Player component type. |
| getTransformComponentType() | ComponentType | O(1) | Provides access to the core TransformComponent type. |
| getVelocityComponentType() | ComponentType | O(1) | Provides access to the core Velocity component type. |

## Integration Patterns

### Standard Usage

A typical plugin will interact with the EntityModule during its own setup phase to register a custom entity.

```java
// In another plugin's setup() method:

// 1. Get the singleton instance
EntityModule entityModule = EntityModule.get();

// 2. Register a custom entity class
entityModule.registerEntity(
    "myplugin:custom_mob",
    CustomMob.class,
    (world) -> new CustomMob(world),
    CustomMob.CODEC
);

// 3. Get a handle to a core component for use in a custom system
ComponentType<EntityStore, TransformComponent> transformType = entityModule.getTransformComponentType();
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new EntityModule()`. The server's plugin framework is solely responsible for its lifecycle. Always use the static `EntityModule.get()` method.
-   **Late Registration:** Do not call `registerEntity` after the server has completed its startup sequence. The ECS is not designed for dynamic type registration at runtime, and this can cause race conditions or inconsistent world state.
-   **Accessing Components Before Setup:** Do not attempt to retrieve a ComponentType from the module before its `setup` method has been executed by the server. This will result in a NullPointerException. Plugin dependencies should be used to enforce the correct initialization order.

## Data Pipeline

The EntityModule is not a data processing component in a traditional pipeline. Instead, it is the **architect of the pipeline itself**. Its primary role in the data flow is during the server's initialization phase.

> **Setup Flow:**
> Server Bootstrap -> Plugin Loader creates **EntityModule** -> Server calls `EntityModule.setup()` -> **EntityModule** registers hundreds of Components and Systems with `EntityStore.REGISTRY` -> World instances are created with a fully configured `EntityStore`.

Once the server is running, the systems registered by EntityModule form the core of the game loop's data processing pipeline for every World.

> **Runtime Entity Update Flow:**
> Game Tick -> `World.tick()` -> `EntityStore.tick()` -> **Systems registered by EntityModule** (e.g., `PlayerProcessMovementSystem`, `RepulsionTicker`, `EntityTrackerSystems`) execute in a dependency-sorted order -> Entity components are mutated -> State changes are sent to clients.

