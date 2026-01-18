---
description: Architectural reference for Damage
---

# Damage

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Transient Event Object

## Definition
```java
// Signature
public class Damage extends CancellableEcsEvent implements IMetaStore<Damage> {
```

## Architecture & Concepts

The Damage class is not a long-lived service but a transient data contract that represents a single instance of a damage event within the server's Entity Component System (ECS). Its primary function is to act as a message bus, carrying a comprehensive payload of information from the system that initiates damage to the systems that process and apply it.

As a subclass of CancellableEcsEvent, any Damage instance is part of a larger event-driven pipeline. Systems can listen for these events, inspect their payload, modify their properties (such as the damage amount), or cancel the event entirely to prevent the damage from being applied.

The implementation of IMetaStore is a critical architectural feature. It transforms the Damage object into a highly extensible data container. Instead of a rigid class with numerous optional fields, the MetaStore allows any system to attach arbitrary, type-safe metadata using a registered MetaKey. This pattern keeps the core Damage API clean while enabling complex interactions, such as attaching knockback profiles, particle effect definitions, or camera shake parameters to a specific damage event.

Furthermore, the Damage.Source interface employs the **Strategy Pattern**. It delegates the responsibility of identifying the damage originator and formatting a corresponding death message to a specific Source implementation (e.g., EntitySource, CommandSource). This decouples the Damage event from the semantics of its origin, allowing for flexible and clear attribution of damage from entities, the environment, or server commands.

## Lifecycle & Ownership

-   **Creation:** A Damage object is instantiated at the moment a damage-causing event occurs. This is typically done within a gameplay system, such as a combat handler processing a weapon hit, a physics system calculating fall damage, or a command executor applying administrative damage. It is always created with `new Damage(...)`.
-   **Scope:** The lifecycle of a Damage object is extremely short, confined to the duration of a single game tick's event processing phase. It is created, dispatched to the ECS event bus, processed synchronously by all relevant listener systems, and then immediately becomes eligible for garbage collection.
-   **Destruction:** The object is managed by the Java garbage collector. No explicit cleanup is required. Once the event dispatch is complete and all references are dropped, the object is destroyed.

## Internal State & Concurrency

-   **State:** The Damage object is highly **mutable** by design. Key properties, particularly the damage amount, are intended to be modified by intermediary systems in the processing chain (e.g., armor systems reducing damage, effect systems amplifying it). The internal MetaStore is also mutable, allowing systems to add or read contextual data as the event propagates.
-   **Thread Safety:** This class is **not thread-safe** and must not be considered as such. All operations on a Damage instance are expected to occur synchronously on the main server thread during the event processing phase of a single game tick. Caching a reference to a Damage object for use in a later tick or on a different thread will lead to severe race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAmount() | float | O(1) | Retrieves the current damage value, which may have been modified by other systems. |
| setAmount(float) | void | O(1) | Modifies the damage value. This is the primary mechanism for armor and resistance calculations. |
| getSource() | Damage.Source | O(1) | Returns the source object that defines the origin of the damage. |
| getDeathMessage(...) | Message | O(N) | Delegates death message generation to the current Source object. Complexity depends on the Source implementation. |
| getMetaStore() | IMetaStoreImpl | O(1) | Provides access to the underlying metadata store for attaching or retrieving extended data. |

## Integration Patterns

### Standard Usage

The standard pattern involves creating a Damage instance, populating it with source, cause, and amount, attaching any relevant metadata, and dispatching it through the world's ECS event bus.

```java
// A combat system calculates a hit on a target entity.
void applyMeleeDamage(Ref<EntityStore> attacker, Ref<EntityStore> target, float baseDamage) {
    // 1. Create the Source and the Damage event object.
    Damage.Source damageSource = new Damage.EntitySource(attacker);
    Damage damageEvent = new Damage(damageSource, DamageCauses.GENERIC, baseDamage);

    // 2. Attach optional, contextual metadata.
    KnockbackComponent knockback = createKnockbackProfile();
    damageEvent.set(Damage.KNOCKBACK_COMPONENT, knockback);
    damageEvent.set(Damage.IMPACT_SOUND_EFFECT, new Damage.SoundEffect(SoundEvents.PLAYER_HURT));

    // 3. Dispatch the event to the ECS for processing by other systems.
    world.getEventBus().post(damageEvent, target);

    // The damageEvent object should now be considered out of scope.
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not cache and reuse a Damage object. Each instance represents a unique event and must be created anew. Reusing an instance will carry over stale state (amount, metadata, cancellation status) and cause unpredictable gameplay bugs.
-   **Asynchronous Modification:** Do not hold a reference to a Damage event for processing on another thread or in a future tick. The object's state is only coherent during the synchronous, single-tick event dispatch chain.
-   **Stateful Listeners:** A system listening for Damage events should not store the event object in one of its own fields. It must fully process the event within the listener method and not retain a reference to it.

## Data Pipeline

The Damage object is the central payload that moves through the server's damage calculation pipeline. Its journey is synchronous and contained within a single game tick.

> Flow:
> Damage Initiator (e.g., Combat System) -> **new Damage(...)** -> ECS Event Bus -> Intercepting Systems (Armor, Status Effects, Enchantments) -> Final Health System -> Entity State Update

