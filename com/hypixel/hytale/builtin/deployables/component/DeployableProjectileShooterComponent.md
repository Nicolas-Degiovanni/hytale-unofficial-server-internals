---
description: Architectural reference for DeployableProjectileShooterComponent
---

# DeployableProjectileShooterComponent

**Package:** com.hypixel.hytale.builtin.deployables.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class DeployableProjectileShooterComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DeployableProjectileShooterComponent is a server-side, data-only component within Hytale's Entity Component System (ECS) architecture. It does not contain any logic itself. Instead, it serves as a state container for entities that have the ability to fire projectiles, such as automated turrets or other placed defenses.

This component's primary responsibility is to track the state associated with projectile management for a single entity:
-   The set of currently active projectiles spawned by this entity.
-   The entity's current target.
-   A list of projectiles marked for cleanup.

Logic that operates on this component is handled by server-side *Systems*. For example, an `DeployableAISystem` would be responsible for acquiring a target and setting it via `setActiveTarget`. A separate `DeployableShootingSystem` would then read this component, and if a target is present, call `spawnProjectile` to request the creation of a new projectile entity. This separation of data (Component) from logic (System) is a core tenet of the ECS pattern.

The use of `Ref<EntityStore>` for entity references is a critical design choice, preventing direct object pointers and allowing the engine to safely manage entity lifecycles without risking memory leaks or dangling references.

## Lifecycle & Ownership
-   **Creation:** This component is instantiated and attached to an entity by the server's core logic, typically during the entity's spawning process. It is never created directly by game script or plugin logic.
-   **Scope:** The lifecycle of a DeployableProjectileShooterComponent is strictly bound to the lifecycle of the entity it is attached to. It persists as long as the parent entity exists in the world.
-   **Destruction:** When the parent entity is destroyed (e.g., a turret is broken), the ECS framework automatically removes and garbage-collects this component along with it.

**WARNING:** The `clone()` method returns `this`, indicating that this component is not designed to be used as a template or value object. It is a unique, stateful instance for each entity.

## Internal State & Concurrency
-   **State:** This component is highly mutable. Its internal lists (`projectiles`, `projectilesForRemoval`) and target reference (`activeTarget`) are expected to change frequently during the game simulation tick. It acts as a live state tracker for the entity's shooting capabilities.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. All reads and writes to this component **must** be performed on the main server game thread. The `spawnProjectile` method's use of a `CommandBuffer` and a world `execute` lambda is a strong indicator of this design pattern. Asynchronous operations or modifications from other threads (e.g., network threads) will lead to `ConcurrentModificationException`, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component. |
| spawnProjectile(...) | void | O(1) | Enqueues a command to spawn a projectile. Does not block or spawn immediately. |
| getProjectiles() | List | O(1) | Returns a direct reference to the list of active projectiles. |
| getProjectilesForRemoval() | List | O(1) | Returns a direct reference to the list of projectiles pending deletion. |
| getActiveTarget() | Ref | O(1) | Gets the current entity target. |
| setActiveTarget(target) | void | O(1) | Sets the current entity target. |

## Integration Patterns

### Standard Usage
This component is intended to be queried and manipulated by server-side systems. A system will get the component from an entity, read its state, and enqueue commands for state changes.

```java
// Example from within a server-side System's update loop
void processShooter(Ref<EntityStore> entityRef, CommandBuffer<EntityStore> commands) {
    DeployableProjectileShooterComponent shooter = entityRef.get(DeployableProjectileShooterComponent.getComponentType());
    Ref<EntityStore> target = shooter.getActiveTarget();

    if (target != null && canFire()) {
        // ... calculate spawn position and direction
        shooter.spawnProjectile(entityRef, commands, projectileConfig, ownerUuid, spawnPos, direction);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new DeployableProjectileShooterComponent()`. Components must be added to entities via the `EntityStore` or a world manager to be correctly registered within the ECS.
-   **Cross-Thread Modification:** Do not get this component and modify its state from a network callback, an asynchronous task, or any thread other than the main server tick thread. All interactions must be marshaled to the main thread, typically by submitting a task to the world scheduler.
-   **External List Modification:** While the API returns direct references to its internal lists, external code should not arbitrarily add or remove elements. These lists are managed by dedicated systems (e.g., a `ProjectileLifecycleSystem`). Unmanaged modification will break state tracking.

## Data Pipeline
This component acts as a stateful junction in the data flow of an AI-controlled entity. It does not process data itself but holds the state that systems use to make decisions.

> Flow:
> AI Target-Acquisition System -> `setActiveTarget()` -> **DeployableProjectileShooterComponent** (State is updated) -> Firing Logic System (Reads component state) -> `spawnProjectile()` -> `CommandBuffer` -> World Engine (Processes command) -> New Projectile Entity Created

