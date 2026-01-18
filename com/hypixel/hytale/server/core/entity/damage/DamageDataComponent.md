---
description: Architectural reference for DamageDataComponent
---

# DamageDataComponent

**Package:** com.hypixel.hytale.server.core.entity.damage
**Type:** Component

## Definition
```java
// Signature
public class DamageDataComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The **DamageDataComponent** is a server-side, data-only component within the Entity-Component-System (ECS) architecture. Its sole responsibility is to store temporal state related to an entity's combat activities. It does not contain any game logic.

This component acts as a shared memory block for various combat-related systems. For example, a **DamageSystem** would write to this component upon a successful hit, while a **HealthRegenSystem** would read from it to determine if an entity is "in combat" and should therefore have its regeneration paused. It effectively decouples the systems that cause combat state changes from the systems that react to them.

Its association with **EntityStore** via the **Component** interface signifies that it is managed and owned by the core entity storage system on the server.

## Lifecycle & Ownership
- **Creation:** A **DamageDataComponent** is never instantiated directly. It is attached to an Entity by a higher-level system, typically the **EntityModule** or an entity factory, when an entity is first created from a prefab or when it first engages in a combat-related action. The presence of the **clone** method indicates its use in entity duplication and templating.

- **Scope:** The lifecycle of a **DamageDataComponent** is strictly bound to the lifecycle of its parent Entity. It persists as long as the Entity exists within the world's **EntityStore**.

- **Destruction:** The component is marked for garbage collection when its parent Entity is removed from the **EntityStore**. There is no manual destruction method; its memory is reclaimed automatically.

## Internal State & Concurrency
- **State:** The internal state is entirely mutable. All fields are designed to be frequently updated by game systems during runtime to reflect the current combat status of the entity. It serves as a volatile snapshot of recent combat events.

- **Thread Safety:** This component is **not thread-safe**. It is a plain Java object with no internal locking or synchronization mechanisms. It is designed to be accessed and mutated exclusively by systems operating on the main server game loop thread.

    **Warning:** Concurrent access from multiple threads without external, system-level locking will lead to race conditions, inconsistent state, and server instability. All interactions with this component must be synchronized by the calling thread, typically by ensuring all logic runs on the main tick.

## API Surface
The public API consists primarily of simple data accessors. The following methods are the primary mutation points for game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLastCombatAction(Instant) | void | O(1) | Updates the timestamp of the last known combat event. Used to manage "in-combat" status flags. |
| setLastDamageTime(Instant) | void | O(1) | Updates the timestamp of the last time the entity received damage. |
| setLastChargeTime(Instant) | void | O(1) | Records the start time of a charged attack or ability. Can be null. |
| setCurrentWielding(WieldingInteraction) | void | O(1) | Sets the current wielding interaction, such as a sword swing. Can be null. |

## Integration Patterns

### Standard Usage
This component must be retrieved from a valid Entity instance. Systems should query for entities and then request the component from them.

```java
// A hypothetical system processing a damage event
void applyDamage(Entity target, DamageInfo damage) {
    // Correct: Retrieve the component from the entity
    DamageDataComponent combatState = target.getComponent(DamageDataComponent.class);

    if (combatState != null) {
        Instant now = Instant.now();
        combatState.setLastDamageTime(now);
        combatState.setLastCombatAction(now);
    }
    
    // ... apply health changes
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DamageDataComponent()`. The component will not be registered with the entity or its managing systems, leading to a "ghost" component that is ignored by the engine.

- **State Caching:** Do not retrieve a component and hold a reference to it across multiple game ticks. The parent entity could be destroyed, leaving a stale reference and causing NullPointerExceptions or use-after-free bugs. Always re-query the component from the entity on each tick you need to access it.

- **Cross-Thread Mutation:** Do not pass an entity or its **DamageDataComponent** to another thread (e.g., for async pathfinding or IO) and modify it there. This will break the single-threaded assumption of the game loop.

## Data Pipeline
The **DamageDataComponent** is a stateful node in the combat data flow, not a processing stage. It is written to by action-oriented systems and read from by state-reactive systems.

> Flow:
> Player Input -> Network Packet -> **InteractionSystem** -> Writes to **DamageDataComponent**
>
> **Game Tick** -> **HealthRegenSystem** -> Reads from **DamageDataComponent** -> Pauses/Resumes Health Regen

