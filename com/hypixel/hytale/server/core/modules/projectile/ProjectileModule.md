---
description: Architectural reference for ProjectileModule
---

# ProjectileModule

**Package:** com.hypixel.hytale.server.core.modules.projectile
**Type:** Singleton

## Definition
```java
// Signature
public class ProjectileModule extends JavaPlugin {
```

## Architecture & Concepts
The ProjectileModule is a foundational server plugin responsible for the entire lifecycle, physics, and interaction logic of projectile entities. As a core module, it establishes the necessary Entity-Component-System (ECS) infrastructure for all projectiles, acting as both a central registry and a factory.

It integrates with the server's core ECS by registering several key components and systems:
*   **Components:** It defines what a projectile *is* by registering `Projectile`, `PredictedProjectile`, and `StandardPhysicsProvider` components. These components are attached to entities to grant them projectile behaviors.
*   **Systems:** It registers `StandardPhysicsTickSystem` and `PredictedProjectileSystems.EntityTrackerUpdate`, which contain the logic to update the state of projectile entities each server tick. This includes calculating trajectories, handling collisions, and synchronizing state with clients.
*   **Factory:** The module provides the canonical `spawnProjectile` method. This is the exclusive, authoritative entry point for creating projectile entities in the world. It abstracts the complex process of assembling an entity with all required components, such as `TransformComponent`, `Velocity`, `ModelComponent`, and `BoundingBox`.

This module is a dependency for any game logic that involves ranged combat or physics-based object launching. It sits between high-level game logic (e.g., a player shooting a bow) and low-level physics and entity management systems.

## Lifecycle & Ownership
-   **Creation:** The ProjectileModule is instantiated once by the server's `PluginLoader` during the server bootstrap sequence. The static singleton instance is set within the constructor, ensuring a single, globally accessible point of control.
-   **Scope:** The instance persists for the entire runtime of the server. It is not world-specific and manages projectiles across the entire server instance.
-   **Destruction:** The module is destroyed and its resources are released only during a full server shutdown.

## Internal State & Concurrency
-   **State:** The module's internal state consists of the singleton `instance` and several `ComponentType` references. These references are initialized once during the single-threaded `setup` phase and are immutable thereafter. The module itself is stateless concerning active projectiles; all per-projectile state is stored in components on the respective entities.
-   **Thread Safety:** This class is thread-safe. The public `spawnProjectile` method is designed for safe concurrent access by requiring a `CommandBuffer`. This pattern defers all entity and component mutations to a queue, which is later processed synchronously by the main game loop. This prevents race conditions and ensures world state integrity.

    **WARNING:** While the method is safe to call from any thread, the provided `CommandBuffer` must be one that is managed and flushed by the primary server thread. Failure to do so will result in lost or incorrectly processed entity creation commands.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ProjectileModule | O(1) | Statically retrieves the singleton instance of the module. |
| spawnProjectile(...) | Ref<EntityStore> | O(C) | Factory method to create and schedule a new projectile entity for spawning. Assembles all necessary components based on the provided configuration. Returns a reference to the entity being created. |
| getProjectileComponentType() | ComponentType | O(1) | Returns the component type for the core Projectile component. |

## Integration Patterns

### Standard Usage
To spawn a projectile, other systems must first retrieve the singleton instance of the module and then invoke the `spawnProjectile` factory method, passing a `CommandBuffer` obtained from the current execution context.

```java
// Obtain the module instance
ProjectileModule projectileModule = ProjectileModule.get();

// Assume 'commandBuffer', 'creator', 'config', 'pos', and 'dir' are available
// from the current game logic context.
Ref<EntityStore> newProjectile = projectileModule.spawnProjectile(
    creator,
    commandBuffer,
    projectileConfig,
    spawnPosition,
    spawnDirection
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ProjectileModule()`. The server's plugin lifecycle manages its creation. Use the static `ProjectileModule.get()` method to access the instance.
-   **Premature Access:** Do not attempt to access the module during early server initialization before the plugin `setup` methods have been called. This will result in a `NullPointerException` when calling `get()`.
-   **State Modification:** Do not attempt to modify the internal state of the module. It is designed to be a stateless service provider after its initial setup.

## Data Pipeline
The `spawnProjectile` method orchestrates a complex data pipeline to transform a high-level request into a fully-realized entity within the ECS world.

> Flow:
> Game Logic Request -> **spawnProjectile(config, pos, dir)** -> Entity Holder Assembly -> Component Calculation (Transform, Velocity, Model) -> CommandBuffer.addEntity -> ECS World Update -> New Projectile Entity

