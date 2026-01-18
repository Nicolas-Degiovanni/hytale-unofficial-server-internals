---
description: Architectural reference for DamageEventSystem
---

# DamageEventSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** System Component (Abstract Base)

## Definition
```java
// Signature
public abstract class DamageEventSystem extends EntityEventSystem<EntityStore, Damage> {
```

## Architecture & Concepts
The DamageEventSystem is an abstract base class that forms the foundation of the server-side damage processing pipeline. It operates within the server's Entity Component System (ECS) architecture, specifically designed to standardize the handling of all Damage events.

This class acts as a specialized filter and dispatcher. By extending EntityEventSystem and hard-coding the event type to Damage, it establishes a strict contract for all subclasses. Any system that needs to react to or modify damage—such as applying armor reductions, triggering status effects, or calculating fall damage—must inherit from this class. This pattern ensures that all damage-related logic is discoverable, ordered, and operates on a consistent data context, the EntityStore.

It is a critical component for decoupling the cause of damage from its effects. A system that *causes* damage simply fires a generic Damage event, without needing to know how that damage will be processed by the various game mechanics.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses (e.g., PlayerDamageSystem, EnvironmentalDamageSystem) are discovered and instantiated by the server's module loader or a System Registry during world initialization. The protected constructor enforces this pattern.
- **Scope:** An instance of a concrete DamageEventSystem subclass has a lifecycle scoped to the world it operates on. It persists as long as the world is loaded and active.
- **Destruction:** The system is destroyed and eligible for garbage collection when its parent world is unloaded or the server shuts down. The System Registry manages this teardown process.

## Internal State & Concurrency
- **State:** The DamageEventSystem base class is stateless. Its primary role is to enforce a type contract. Subclasses are expected to be stateful, potentially caching configuration or references to other systems.
- **Thread Safety:** **Not thread-safe.** Like most ECS systems, all interactions with a DamageEventSystem and the associated EntityStore must occur on the main server tick thread. The underlying EntityStore is not designed for concurrent access, and attempting to process events from other threads will lead to data corruption, race conditions, and server instability.

**WARNING:** All subclass implementations must be designed to execute quickly and without blocking, as they are invoked synchronously within the main game loop.

## API Surface
The public contract is defined by its role as a base class for extension, not through direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DamageEventSystem() | protected | O(1) | Constructor for subclasses. Registers the system to listen exclusively for events of type Damage. |

## Integration Patterns

### Standard Usage
The only correct usage is to extend this class to create a new system that processes damage events. The game engine's System Registry will automatically discover and invoke the system.

```java
// Correct: Subclass to implement a specific damage mechanic
public class FireDamageSystem extends DamageEventSystem {

    @Override
    public void process(EntityStore store, Damage damageEvent) {
        // Implement logic for fire-specific damage effects,
        // such as checking for fire resistance components on the target entity.
        if (damageEvent.getSource().isFire()) {
            // ... modify components in the EntityStore
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance via `new`. The system is abstract and managed by the engine.
- **Incorrect Event Handling:** Do not extend this class to handle events other than Damage. Its entire purpose is hard-wired to the Damage event type.
- **State Modification Outside of `process`:** Avoid modifying the EntityStore from any method other than the `process` method to maintain a clear and predictable data flow within the server tick.

## Data Pipeline
This system is a processor in the server's event-driven data flow. It receives Damage events and translates them into state changes within the EntityStore.

> Flow:
> Game Action (e.g., Arrow Hit) -> Damage Event Published -> Event Bus -> **DamageEventSystem Subclass** -> EntityStore Component Modification -> Client State Update

