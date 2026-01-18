---
description: Architectural reference for EntityEvent
---

# EntityEvent

**Package:** com.hypixel.hytale.server.core.event.events.entity
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class EntityEvent<EntityType extends Entity, KeyType> implements IEvent<KeyType> {
```

## Architecture & Concepts
The EntityEvent class is an abstract foundation for all events within the server core that are directly associated with a game Entity. It is not a concrete, dispatchable event itself, but rather a contract that guarantees any subclass will provide a reference to the relevant Entity.

This class is a cornerstone of the server's event-driven architecture. It decouples high-level game systems (e.g., AI, Combat, Quests) from the low-level mechanics that trigger state changes. For example, a system that handles player health does not need to know *what* caused the damage; it only needs to listen for an `EntityDamageEvent` (a subclass of EntityEvent) and can retrieve the affected entity from the event payload.

By enforcing a generic constraint where `EntityType` must extend `Entity`, the system provides compile-time type safety for event listeners, eliminating the need for unsafe casting.

### Lifecycle & Ownership
-   **Creation:** An EntityEvent is never instantiated directly due to its abstract nature. Concrete subclasses (e.g., `EntitySpawnEvent`, `EntityDeathEvent`) are created by various server systems at the moment a relevant action occurs. For instance, the `WorldManager` would instantiate an `EntitySpawnEvent` when a new creature is added to a zone.
-   **Scope:** Transient and extremely short-lived. An EntityEvent object exists only for the duration of its dispatch through the central `EventBus`. It acts as a temporary, immutable data carrier.
-   **Destruction:** Once all registered listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There are no manual cleanup or `close` methods. Holding a reference to an event object after its dispatch cycle is complete is considered a memory leak.

## Internal State & Concurrency
-   **State:** Immutable. The internal `entity` field is declared `final` and is assigned only once during construction. This design is critical for a stable event system, as it prevents one event listener from modifying the event payload and causing unpredictable side effects for subsequent listeners in the chain.

-   **Thread Safety:** The EntityEvent object itself is thread-safe due to its immutability. However, the `Entity` object it contains is almost certainly mutable and not thread-safe.

    **WARNING:** Event dispatch is typically synchronized with the main server game loop (the "tick"). Listeners should operate under the assumption that they are executing on this main thread. Modifying the state of the contained `Entity` from an asynchronous task or a different thread without proper synchronization will lead to severe concurrency issues, data corruption, and server instability.

## API Surface
The public contract is minimal, focusing exclusively on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntity() | EntityType | O(1) | Returns the root entity associated with this event. This value is guaranteed to be non-null. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a concrete subclass of EntityEvent for a specific game action and dispatch it through the server's event bus. Listeners then subscribe to this specific event type.

```java
// 1. Define a concrete event
public class EntityHealedEvent extends EntityEvent<Player, World> {
    private final double amount;

    public EntityHealedEvent(Player player, double amount) {
        super(player);
        this.amount = amount;
    }

    public double getAmount() {
        return this.amount;
    }
}

// 2. Dispatch the event from game logic
Player targetPlayer = world.getPlayer("notch");
double healAmount = 50.0;
server.getEventBus().post(new EntityHealedEvent(targetPlayer, healAmount));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Attempting to create an instance with `new EntityEvent()` is a compile-time error. You must always extend the class.
-   **Modifying Contained State:** Do not use the event as a mechanism to pass mutable state between listeners. A listener that modifies the `Entity` object can create non-obvious dependencies and race conditions. Events report facts; they should not be used as remote procedure calls.
-   **Holding References:** Storing an instance of an EntityEvent in a long-lived object after the event has been processed. This prevents garbage collection and serves no logical purpose, as the event represents a point-in-time snapshot of a past occurrence.

## Data Pipeline
The EntityEvent serves as a standardized data carrier in the flow from game action to system reaction.

> Flow:
> Game Action (e.g., Player uses a potion) -> Game Logic instantiates `EntityHealedEvent` -> EventBus dispatches event -> **EntityEvent (as superclass)** provides `getEntity()` to listeners -> Listener (e.g., `ParticleEffectSystem`) reads the entity's location -> System spawns healing particles at the entity.

