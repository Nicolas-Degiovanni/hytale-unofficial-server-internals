---
description: Architectural reference for ProjectileComponent
---

# ProjectileComponent

**Package:** com.hypixel.hytale.server.core.entity.entities
**Type:** Data Component

## Definition
```java
// Signature
public class ProjectileComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ProjectileComponent is a data-oriented component within the server's Entity Component System (ECS). It does not represent an entity itself, but rather encapsulates the state, configuration, and core logic for an entity that behaves as a projectile.

This component acts as the central coordinator for a projectile's lifecycle. It holds configuration loaded from a corresponding Projectile asset, identified by *projectileAssetName*. It delegates complex physics calculations, including movement and collision detection, to an internal instance of SimplePhysicsProvider. Upon physics events like collisions or bounces, the SimplePhysicsProvider invokes callback handlers within this component (impactHandler, bounceHandler), which then trigger game logic such as applying damage, playing sounds, and spawning particles.

It is fundamentally a state machine, transitioning from *in-flight* to *impacted* and finally to *death*, orchestrating effects and game consequences at each stage. It is designed to be attached to an Entity alongside other components like TransformComponent, Velocity, and DespawnComponent, which collectively define a complete projectile.

## Lifecycle & Ownership
- **Creation:** A ProjectileComponent is not instantiated directly. The canonical creation path is through the static factory method **assembleDefaultProjectile**. This method constructs a new Entity holder and attaches a ProjectileComponent along with other essential components (DespawnComponent, TransformComponent), ensuring the entity is valid and ready to be spawned into the world.
- **Scope:** The component's lifetime is strictly tied to the entity it is attached to. It persists from the moment it is fired via the *shoot* method until it is destroyed.
- **Destruction:** The component is destroyed when its parent entity is removed from the EntityStore. This typically occurs when the projectile hits a target, its despawn timer expires (managed by DespawnComponent), or it is manually removed from the world. The **onProjectileDeath** method is invoked just before destruction to play any final visual or audio effects.

## Internal State & Concurrency
- **State:** The component's state is highly **mutable**. Internal fields such as *deadTimer*, *haveHit*, and the state within the delegated SimplePhysicsProvider are continuously updated by various game systems throughout the projectile's flight. It also holds a transient, immutable reference to the Projectile asset configuration after initialization.
- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed and modified exclusively by the main server thread that processes the entity update loop. The use of thread-local resources in its methods (e.g., SpatialResource.getThreadLocalReferenceList) reinforces this design. Unsynchronized access from other threads will lead to race conditions, data corruption, and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assembleDefaultProjectile(...) | Holder<EntityStore> | O(1) | Static factory. The only correct way to create a new projectile entity. |
| initialize() | boolean | O(1) | Links the component to its asset configuration. Must be called before use. |
| initializePhysics(bbox) | void | O(1) | Configures the internal physics provider with the entity's bounding box. |
| shoot(...) | void | O(1) | Initiates the projectile's flight, setting its initial velocity and creator. |
| consumeDeadTimer(dt) | boolean | O(1) | Ticks down the post-impact timer. Returns true when the timer expires. |
| onProjectileDeath(...) | void | O(N) | Triggers final effects like explosions and particles before the entity is removed. |

## Integration Patterns

### Standard Usage
The component should be created via its factory method, configured with the *shoot* method, and then the resulting entity holder should be added to the world for processing by the game's systems.

```java
// How a developer should normally use this
// Assume 'world' and 'time' resources are available
Vector3d startPosition = new Vector3d(10, 64, 10);
Vector3f direction = new Vector3f(0, 0); // Yaw, Pitch

// 1. Create the projectile entity using the factory
Holder<EntityStore> projectileHolder = ProjectileComponent.assembleDefaultProjectile(
    time, 
    "hytale:arrow", // Asset name
    startPosition, 
    direction
);

// 2. Retrieve the component to configure its launch
ProjectileComponent projectileComp = projectileHolder.getComponent(ProjectileComponent.getComponentType());
projectileComp.shoot(
    projectileHolder,
    creatorEntityUUID, // The UUID of the entity that fired it
    startPosition.x,
    startPosition.y,
    startPosition.z,
    direction.x,
    direction.y
);

// 3. Add the fully configured entity to the world
world.getEntityStore().addEntity(projectileHolder);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ProjectileComponent()`. This bypasses the `assembleDefaultProjectile` factory, resulting in an incomplete entity that lacks critical components like TransformComponent and DespawnComponent, causing system failures.
- **Pre-Initialization Access:** Do not attempt to access the internal Projectile asset (via getProjectile) before the `initialize` method has been called by the engine. This will result in a NullPointerException.
- **External Physics Manipulation:** Do not modify the Velocity component on the entity directly after the projectile has been shot. All physics state should be managed internally by the ProjectileComponent's SimplePhysicsProvider. Direct manipulation will break collision and movement predictions.

## Data Pipeline
The ProjectileComponent follows an event-driven state flow rather than a linear data pipeline.

> Flow:
> 1. **Creation:** `assembleDefaultProjectile` is called, creating an entity with this component in an *idle* state.
> 2. **Launch:** The `shoot` method is called, setting the creator and initial velocity. The state transitions to *in-flight*.
> 3. **Update Tick (Loop):** The Physics System updates the entity's position based on velocity managed by the internal SimplePhysicsProvider.
> 4. **Collision Event:** The SimplePhysicsProvider detects a collision and invokes a callback.
>    - **Bounce:** `bounceHandler` is called -> `onProjectileBounce` plays effects. The state remains *in-flight*.
>    - **Impact:** `impactHandler` is called -> `onProjectileHitEvent` or `onProjectileMissEvent` is triggered. Damage is applied, effects are played, and the *deadTimer* is activated. The state transitions to *impacted*.
> 5. **Post-Impact Tick (Loop):** The Entity Update System calls `consumeDeadTimer`.
> 6. **Death:** When the timer expires, `onProjectileDeath` is called to trigger final explosions or effects. The entity is then removed from the world by the Despawn system.

