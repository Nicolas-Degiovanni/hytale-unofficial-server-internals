---
description: Architectural reference for DeployableProjectileComponent
---

# DeployableProjectileComponent

**Package:** com.hypixel.hytale.builtin.deployables.component
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class DeployableProjectileComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DeployableProjectileComponent is a data-only component within the server-side Entity-Component-System (ECS) architecture. Its sole purpose is to attach state to an entity, marking it as a projectile and storing its position from the previous game tick.

This component contains no logic. It serves as a data container for systems, primarily the physics and collision detection systems. By comparing the value in **previousTickPosition** with the entity's current position (stored in a separate PositionComponent), a system can accurately calculate the entity's trajectory over the last tick. This is fundamental for implementing continuous collision detection (CCD), preventing fast-moving projectiles from passing through thin objects between discrete physics updates.

This component is a foundational piece of the deployables feature, enabling items like grenades, snowballs, or other thrown objects to interact realistically with the game world.

### Lifecycle & Ownership
- **Creation:** This component is never instantiated directly. It is created and attached to an Entity by a higher-level system when a projectile is spawned in the world. For example, a weapon system, upon firing a projectile, will construct the projectile Entity and add a DeployableProjectileComponent to it.
- **Scope:** The lifecycle of this component is strictly bound to the Entity it is attached to. It persists as long as the projectile entity exists within the server's EntityStore.
- **Destruction:** The component is destroyed automatically when its parent Entity is removed from the world. This occurs when the projectile collides with a surface, its lifetime expires, or it is otherwise despawned by a game system.

## Internal State & Concurrency
- **State:** The component's state is mutable. It holds a single Vector3d, **previousTickPosition**, which is updated by a dedicated physics system at the end of each game tick. The class employs defensive cloning in its public accessors to ensure state integrity and prevent external objects from holding a direct reference to its internal vector.
- **Thread Safety:** **This component is not thread-safe.** Like most ECS components, it is designed to be accessed and manipulated exclusively by systems operating on the main server game loop thread. Unsynchronized access from other threads (e.g., network or worker threads) will result in severe race conditions, leading to unpredictable physics and potential server instability. All interactions must be scheduled and executed on the main tick.

## API Surface
The public API is minimal, focusing entirely on state management for the owning system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, static type identifier for this component class. |
| clone() | Component | O(1) | Creates a deep copy of the component, including a new instance of the position vector. |
| getPreviousTickPosition() | Vector3d | O(1) | Returns a defensive clone of the position from the previous tick. |
| setPreviousTickPosition(pos) | void | O(1) | Updates the stored position. The input vector is cloned to maintain ownership. |

## Integration Patterns

### Standard Usage
This component is intended to be queried and managed by a server-side system. The system iterates over all entities possessing this component each tick to perform physics calculations.

```java
// Example within a hypothetical ProjectilePhysicsSystem
EntityStore entityStore = server.getWorld().getEntityStore();
View<Entity> projectileView = entityStore.getView(
    ComponentFilter.all(PositionComponent.class, DeployableProjectileComponent.class)
);

for (Entity entity : projectileView) {
    PositionComponent currentPos = entity.getComponent(PositionComponent.class);
    DeployableProjectileComponent projectileData = entity.getComponent(DeployableProjectileComponent.class);

    Vector3d previousPos = projectileData.getPreviousTickPosition();

    // 1. Perform collision detection using the line segment from previousPos to currentPos.
    detectCollisions(previousPos, currentPos.getPosition());

    // 2. After all physics are resolved for the tick, update the component for the next frame.
    projectileData.setPreviousTickPosition(currentPos.getPosition());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DeployableProjectileComponent()`. Components must be added to an entity via the Entity or EntityStore API, such as `entity.addComponent()`. The ECS framework manages the component's lifecycle.
- **External State Mutation:** Do not call `setPreviousTickPosition` from arbitrary game logic. This method should only be called by the primary system responsible for managing projectile physics, typically at the end of a tick update cycle. Uncoordinated writes will break collision detection.
- **Component as a Flag:** Do not add this component to an entity simply to "tag" it as a projectile without a corresponding system to manage its state. An unmanaged DeployableProjectileComponent provides no value and its **previousTickPosition** will become stale.

## Data Pipeline
The data within this component is part of a cyclical, per-tick pipeline driven by the core physics system.

> Flow (within one game tick):
> ProjectileSystem `update()` -> Reads **DeployableProjectileComponent**.previousTickPosition -> Reads Entity's current PositionComponent -> Calculates trajectory vector -> Performs collision checks -> Updates Entity's Position and Velocity -> **(End of Tick)** -> Calls **DeployableProjectileComponent**.setPreviousTickPosition(newPosition)

