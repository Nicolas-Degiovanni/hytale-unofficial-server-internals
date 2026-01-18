---
description: Architectural reference for DeferredCorpseRemoval
---

# DeferredCorpseRemoval

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Transient Component

## Definition
```java
// Signature
public class DeferredCorpseRemoval implements Component<EntityStore> {
```

## Architecture & Concepts
The DeferredCorpseRemoval component is a state-holding data object within the server-side Entity Component System (ECS). Its sole purpose is to act as a countdown timer attached to a deceased entity, commonly referred to as a "corpse".

This component embodies the principle of deferred execution. Instead of removing an entity from the world immediately upon its death, this component is attached, allowing the entity to persist in a static state for a configurable duration. This is crucial for gameplay mechanics such as player looting, visual effects, and preventing the immediate disappearance of defeated enemies.

A dedicated server system, not shown here, is responsible for querying all entities that possess this component. During each server tick, this system invokes the component's *tick* method, advancing its internal timer. When the timer expires, the system initiates the final removal of the entity from the world. This pattern decouples the logic of entity death from the logic of world cleanup, improving modularity and performance.

## Lifecycle & Ownership
- **Creation:** An instance is created and attached to an entity by a higher-level system, typically the DamageModule or a related death-processing system, at the moment an entity's health is reduced to zero or below. The initial duration is passed into the constructor.
- **Scope:** The component's lifetime is finite and directly corresponds to the `timeUntilCorpseRemoval` value provided during its creation. It persists as part of an entity for the duration of this countdown.
- **Destruction:** The component is destroyed implicitly when its parent entity is removed from the EntityStore. This removal is the direct result of this component's `tick` method returning true, which signals to the managing system that the countdown has completed.

## Internal State & Concurrency
- **State:** The component maintains a single piece of mutable state: the `timeRemaining` field. This value is volatile and is decremented on every server tick by the managing system.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be exclusively owned and modified by a single system operating on the main server game loop thread. Any attempt to call the `tick` method or modify `timeRemaining` from an asynchronous task or worker thread will introduce severe race conditions and lead to unpredictable entity cleanup behavior. All interactions must be synchronized with the primary server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the DamageModule. |
| tick(float dt) | boolean | O(1) | Decrements the internal timer by the delta time. Returns true if the timer has expired, false otherwise. |
| clone() | Component | O(1) | Creates a deep copy of the component with the current `timeRemaining`. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly. It is managed by a server-side system that processes entity cleanup. The conceptual logic within such a system would resemble the following.

```java
// Hypothetical CorpseCleanupSystem
for (Entity entity : entityStore.getEntitiesWith(DeferredCorpseRemoval.class)) {
    DeferredCorpseRemoval timer = entity.getComponent(DeferredCorpseRemoval.class);
    
    // tick() returns true when the timer expires
    if (timer.tick(deltaTime)) {
        world.removeEntity(entity.getId());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Ticking:** Do not call the `tick` method from arbitrary application logic. The timing of corpse removal is critical and must be managed exclusively by the designated server system to ensure consistency with the game loop.
- **State Manipulation:** Do not modify the public `timeRemaining` field directly. This bypasses the intended `tick`-based lifecycle and can lead to corpses that never expire or that are removed prematurely.
- **Orphaned Instances:** Do not create an instance of DeferredCorpseRemoval without immediately attaching it to an entity. An unattached component serves no purpose and will be garbage collected without effect.

## Data Pipeline
The flow of data and control for this component is event-driven and managed by the server's core systems.

> Flow:
> Entity Death Event -> Death Handling System -> Attaches **DeferredCorpseRemoval** to Entity -> Corpse Cleanup System (on subsequent server ticks) -> Invokes `tick()` -> `true` return value -> System issues Entity Removal Command -> Entity is removed from EntityStore

