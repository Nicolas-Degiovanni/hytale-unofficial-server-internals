---
description: Architectural reference for ImpactConsumer
---

# ImpactConsumer

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ImpactConsumer {
   void onImpact(
      @Nonnull Ref<EntityStore> var1, @Nonnull Vector3d var2, @Nullable Ref<EntityStore> var3, @Nullable String var4, @Nonnull CommandBuffer<EntityStore> var5
   );
}
```

## Architecture & Concepts
The ImpactConsumer interface is a core component of the server-side projectile system. It embodies the **Strategy** design pattern, decoupling the physics of a projectile (movement, collision detection) from its gameplay effects (damage, explosions, status effects).

Instead of hardcoding projectile behavior, the system requires a concrete implementation of this interface for each projectile type. When a projectile's physics simulation detects a collision, it does not directly modify the world. Instead, it invokes the `onImpact` method of the projectile's configured ImpactConsumer, providing a rich context of the collision event.

This design makes the projectile system highly extensible and data-driven. New projectile behaviors can be defined entirely through configuration and scripting by providing new implementations of this interface, without altering the underlying physics engine. The use of a CommandBuffer as a parameter is critical for ensuring that world-state modifications are deferred and executed in a safe, synchronized phase of the server tick.

## Lifecycle & Ownership
- **Creation:** Implementations are typically defined as lambda expressions or method references within a projectile's configuration data. They are not instantiated by a dependency injection framework but are created alongside the definition of a projectile asset.
- **Scope:** An ImpactConsumer instance is stateless and its lifecycle is tied to the projectile configuration asset it belongs to. It persists in memory as long as the corresponding projectile type is loaded by the server.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated projectile assets, for example, during a server shutdown or when a mod is unloaded.

## Internal State & Concurrency
- **State:** Implementations of ImpactConsumer are **expected to be stateless**. The same instance may be used concurrently for multiple impact events from different projectiles of the same type. Storing mutable state within an implementation is a severe anti-pattern that can lead to unpredictable behavior and race conditions.
- **Thread Safety:** The `onImpact` method may be invoked from a dedicated physics thread, separate from the main server thread. It is **not safe** to directly modify world state (e.g., entity health, block positions) from within this method. All world-modifying actions **must** be queued onto the provided CommandBuffer. The CommandBuffer is a thread-safe mechanism for deferring operations to be executed synchronously on the main game loop, preventing data corruption and concurrency issues.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onImpact(projectile, position, hitEntity, hitZone, commands) | void | O(N) | The single functional method defining the projectile's impact behavior. N is the number of commands queued. |

**Parameters:**
- **projectile:** A reference to the projectile entity that caused the impact.
- **position:** The precise world-space Vector3d coordinate of the collision.
- **hitEntity:** A nullable reference to the entity that was struck. Will be null if the impact was with a block or terrain.
- **hitZone:** A nullable string identifying the specific part of the entity that was hit, such as "head" or "torso".
- **commands:** The CommandBuffer used to safely queue world-state changes for later execution.

## Integration Patterns

### Standard Usage
The intended use is to provide a lambda expression during projectile configuration. This lambda uses the provided context to queue gameplay effects onto the CommandBuffer.

```java
// Example of defining a projectile's behavior
// This is conceptual; actual API may differ.
ProjectileConfig.newBuilder("fireball")
    .withImpactConsumer((projectile, pos, hitEntity, hitZone, commands) -> {
        // Create an explosion effect at the impact point
        commands.spawnEntity("explosion_effect", pos);

        // If an entity was hit, apply damage
        if (hitEntity != null) {
            commands.dealDamage(hitEntity, 20.0, "fire");
        }
    })
    .build();
```

### Anti-Patterns (Do NOT do this)
- **Direct World Modification:** Bypassing the CommandBuffer to directly access and modify the world or entity state. This will break thread safety and lead to server instability or data corruption.
- **Stateful Implementations:** Creating an ImpactConsumer that maintains mutable state across multiple calls. This is not thread-safe and will cause unpredictable gameplay logic.
- **Blocking Operations:** Performing long-running tasks like file I/O, database queries, or complex synchronous pathfinding within `onImpact`. This will stall the server's physics simulation thread, causing severe performance degradation.

## Data Pipeline
The flow of data and control for a projectile impact is strictly defined to ensure stability and performance.

> Flow:
> Physics Engine detects collision -> Projectile System invokes **ImpactConsumer.onImpact()** -> Implementation queues actions on CommandBuffer -> Main Server Tick executes CommandBuffer -> World State is modified

