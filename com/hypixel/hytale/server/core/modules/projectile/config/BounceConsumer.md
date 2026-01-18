---
description: Architectural reference for BounceConsumer
---

# BounceConsumer

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Functional Interface / Callback

## Definition
```java
// Signature
@FunctionalInterface
public interface BounceConsumer {
   void onBounce(@Nonnull Ref<EntityStore> var1, @Nonnull Vector3d var2, @Nonnull CommandBuffer<EntityStore> var3);
}
```

## Architecture & Concepts
The BounceConsumer interface is a behavioral contract that decouples a projectile's core physics simulation from the specific game logic that executes when it bounces. It embodies the Strategy pattern, allowing projectile behavior to be defined via configuration rather than being hardcoded.

When the server's physics engine detects that a projectile entity should bounce off a surface, it does not execute the bounce effects directly. Instead, it invokes the onBounce method on the BounceConsumer instance associated with that projectile's configuration. This allows for a vast range of dynamic effects—a bouncing arrow might simply change trajectory, while a magical orb could trigger explosions, spawn new entities, or apply status effects by issuing commands to the provided CommandBuffer.

This design is central to the server's data-driven entity system, enabling designers to create complex projectile interactions without modifying core engine code.

### Lifecycle & Ownership
- **Creation:** Implementations of BounceConsumer are typically instantiated as lambda expressions or method references during the loading and parsing of projectile configuration assets. They are part of a projectile's immutable template data.
- **Scope:** An instance of a BounceConsumer implementation is stateless and lives as long as the projectile configuration that references it. It is effectively a singleton for a given projectile type.
- **Destruction:** The object is eligible for garbage collection when its associated projectile configuration is unloaded from memory, for instance, when a server shuts down or a game module is reloaded.

## Internal State & Concurrency
- **State:** Implementations of BounceConsumer are expected to be **completely stateless**. All necessary context for executing the bounce logic—the projectile entity, the bounce location, and a command buffer for world modification—is provided through the onBounce method arguments. Storing mutable state within a consumer is a severe anti-pattern that can lead to unpredictable behavior across multiple instances of the same projectile.
- **Thread Safety:** This interface is **not thread-safe**. The onBounce method is designed to be called exclusively from the main server game loop thread during the entity update phase. The provided CommandBuffer is tied to the current tick and must not be accessed, stored, or manipulated from any other thread. Doing so will result in world state corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onBounce(entity, position, commands) | void | O(N) | Executes the game logic for a projectile bounce. The complexity is O(N) where N is the number of commands enqueued into the CommandBuffer. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda or method reference when defining a projectile's behavior, typically within a higher-level configuration system. The implementation uses the provided CommandBuffer to queue up world-modifying actions.

```java
// Example: Defining a projectile that creates an explosion on bounce
ProjectileConfig.Builder builder = new ProjectileConfig.Builder();
builder.withBounceConsumer((entityRef, position, commands) -> {
    // Use the command buffer to safely queue world changes
    commands.spawnEntity(EntityType.EXPLOSION, position);
    commands.playSound(Sound.EXPLODE, position);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct World Modification:** Never attempt to modify the EntityStore or world state directly from within onBounce. All state changes **must** be enqueued through the provided CommandBuffer to prevent concurrent modification exceptions and ensure deterministic tick processing.
- **Blocking Operations:** Do not perform file I/O, network requests, or any other long-running, blocking operations within an implementation. This will stall the server's main tick thread, causing severe performance degradation.
- **Stateful Implementations:** Avoid creating implementations that store mutable instance variables. This breaks the functional nature of the component and can cause unintended side effects between different projectile instances of the same type.

## Data Pipeline
The BounceConsumer acts as a processing node within the server's entity physics pipeline.

> Flow:
> Server Tick -> Physics System Update -> Collision Detection -> **BounceConsumer.onBounce()** -> CommandBuffer -> End-of-Tick Command Execution -> World State Update

