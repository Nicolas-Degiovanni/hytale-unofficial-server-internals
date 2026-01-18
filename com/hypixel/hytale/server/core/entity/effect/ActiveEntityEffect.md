---
description: Architectural reference for ActiveEntityEffect
---

# ActiveEntityEffect

**Package:** com.hypixel.hytale.server.core.entity.effect
**Type:** Transient State Object

## Definition
```java
// Signature
public class ActiveEntityEffect implements Damage.Source {
```

## Architecture & Concepts

The ActiveEntityEffect class is a data-centric object that represents a *live instance* of a status effect applied to a specific entity. It is not a system or manager; rather, it is the runtime state for effects like poison, regeneration, or invulnerability. Each instance corresponds to one effect on one entity.

Architecturally, this class serves as a state machine driven by the server's main game loop. Its behavior is defined by a corresponding static data asset, an EntityEffect, which contains the configuration (e.g., damage amount, stat modifiers). The ActiveEntityEffect holds the dynamic, instance-specific state, such as the remaining duration and time since the last damage tick.

A critical design feature is its implementation of the **Damage.Source** interface. This elevates an ActiveEntityEffect from a simple state container to an actor within the combat system. It can originate damage events, enabling mechanics like Damage-over-Time (DoT) where the effect itself, not another entity, is the cause of harm.

This class is tightly integrated with an Entity-Component-System (ECS) paradigm. Its primary logic method, *tick*, operates on an entity reference (Ref) and queues all state-mutating operations (like dealing damage or changing stats) onto a **CommandBuffer**. This pattern ensures that all changes are resolved deterministically at the end of the game tick, preventing race conditions and maintaining world state integrity.

## Lifecycle & Ownership

-   **Creation:** An ActiveEntityEffect is instantiated whenever a status effect is applied to an entity. This is typically triggered by game events such as a weapon hit, potion consumption, or entering an environmental hazard. A higher-level system, such as an InteractionSystem or EffectSystem, is responsible for its creation and attachment to the target entity.

-   **Scope:** The object's lifetime is bound to the duration of the effect on its host entity. It persists as long as its *remainingDuration* is greater than zero or its *infinite* flag is set. It is owned by a component on the entity, likely a list or map within a hypothetical ActiveEffectsComponent.

-   **Destruction:** The object is marked for garbage collection when its duration expires or it is explicitly dispelled. The managing system is responsible for removing the instance from the entity's component list, at which point it is no longer referenced and can be cleaned up by the JVM.

## Internal State & Concurrency

-   **State:** The internal state of ActiveEntityEffect is highly **mutable**. Its primary function is to track the countdown of *remainingDuration* and the interval timers like *sinceLastDamage*. It is not a cache; it *is* the canonical source of truth for a single, active status effect instance.

-   **Thread Safety:** This class is **not thread-safe** and must not be considered so. It is designed exclusively for single-threaded access within the main server game loop. All methods, particularly *tick*, assume they are called sequentially and without contention. Any concurrent modification from other threads will lead to state corruption and unpredictable game logic.

## API Surface

The public API is minimal, exposing only the core logic for updates and its contract as a damage source.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(commandBuffer, ref, entityEffect, entityStatMap, dt) | void | O(1) | The primary update method. Calculates and queues damage and stat changes for the current game tick. Must be called once per tick by a managing system. |
| getDeathMessage(info, targetRef, componentAccessor) | Message | O(1) | Fulfills the Damage.Source contract. Generates a localized death message if this effect is the cause of an entity's death. |

## Integration Patterns

### Standard Usage

The ActiveEntityEffect is designed to be managed by a dedicated system. This system iterates through all entities with active effects and calls the *tick* method for each one during the server's update phase.

```java
// Hypothetical EffectSystem update loop
for (Entity entity : world.getEntitiesWith(ActiveEffectsComponent.class)) {
    ActiveEffectsComponent effects = entity.get(ActiveEffectsComponent.class);
    float deltaTime = world.getDeltaTime();

    // Iterate backwards to allow for safe removal
    for (int i = effects.list.size() - 1; i >= 0; i--) {
        ActiveEntityEffect effect = effects.list.get(i);
        EntityEffect config = assetManager.get(effect.getEntityEffectIndex());

        // Delegate update logic to the effect instance
        effect.tick(commandBuffer, entity.getRef(), config, entity.get(EntityStatMap.class), deltaTime);

        if (effect.getRemainingDuration() <= 0 && !effect.isInfinite()) {
            effects.list.remove(i); // Effect has expired
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct State Manipulation:** Do not modify fields like *remainingDuration* directly from an external system. To cancel an effect, the managing system should provide a proper API that removes the object from the entity's component list.

-   **Ticking Outside the Game Loop:** Calling the *tick* method outside of the main, synchronized update phase will break game state determinism. All operations must be funneled through the CommandBuffer during the correct simulation step.

-   **Sharing Instances:** An ActiveEntityEffect instance must never be shared between multiple entities. Each entity must have its own unique instance for each effect applied to it.

## Data Pipeline

The ActiveEntityEffect acts as a processor within the main game tick pipeline. It consumes time and produces commands.

> Flow:
> Game Loop Tick -> Effect Management System -> **ActiveEntityEffect.tick()** -> CommandBuffer -> Damage & Stat Systems Resolution -> World State Update

