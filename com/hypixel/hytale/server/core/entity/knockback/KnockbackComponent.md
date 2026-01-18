---
description: Architectural reference for KnockbackComponent
---

# KnockbackComponent

**Package:** com.hypixel.hytale.server.core.entity.knockback
**Type:** Transient Data Component

## Definition
```java
// Signature
public class KnockbackComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The KnockbackComponent is a pure data container within the server-side Entity-Component-System (ECS) architecture. It does not contain any logic itself; instead, it holds the state required to apply a temporary, physics-based impulse to an entity. This component represents a single instance of a knockback effect.

Its primary role is to act as a message between different game systems. For example, a CombatSystem might create and attach a KnockbackComponent to an entity after a weapon hit. On a subsequent server tick, a PhysicsSystem will detect this component, apply its velocity data to the entity's core physics state, and manage its lifecycle.

The component's design, featuring a base velocity, a list of scalar modifiers, and a duration, provides a flexible mechanism for implementing complex knockback behaviors. Modifiers can be added by various sources (e.g., enchantments, status effects) before being collapsed into a final velocity vector by the `applyModifiers` method.

## Lifecycle & Ownership
- **Creation:** A KnockbackComponent is instantiated and attached to an entity dynamically in response to a game event, such as an explosion or a melee attack. This is typically handled by high-level game logic systems (e.g., a CombatSystem). It is never pre-allocated.
- **Scope:** The component's lifetime is explicitly time-bound by its `duration` field. It is owned by the single entity to which it is attached and should never be shared or referenced elsewhere.
- **Destruction:** The component is marked for removal and garbage collected when its internal `timer` exceeds its `duration`. This cleanup is managed by a central entity processing system during the server tick, ensuring that expired knockback effects are purged automatically.

## Internal State & Concurrency
- **State:** The KnockbackComponent is highly mutable. Its fields, such as `velocity` and `timer`, are expected to be updated frequently by the owning system. The `modifiers` list acts as a temporary write buffer which is cleared after its values are consumed by `applyModifiers`.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be created, modified, and read exclusively by the main server game loop thread. Any attempt to access or mutate an instance from an asynchronous task or network thread will result in severe concurrency issues, including race conditions and unpredictable physics calculations. All interactions must be synchronized with the server tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type identifier for this component from the EntityModule. |
| setVelocity(Vector3d velocity) | void | O(1) | Sets the base velocity vector for the knockback impulse. |
| addModifier(double modifier) | void | O(1) | Appends a scalar multiplier to the internal modifiers list. |
| applyModifiers() | void | O(N) | Applies all stored modifiers to the velocity vector by sequential multiplication, then clears the list. |
| incrementTimer(float time) | void | O(1) | Advances the internal lifetime timer. **Warning:** For internal system use only. |
| clone() | Component | O(1) | Creates a shallow copy of the component. **Warning:** The internal Vector3d is copied by reference, not value. |

## Integration Patterns

### Standard Usage
The component should be created, configured, and attached to an entity by a system handling a specific game event. The core physics engine will then automatically detect and process it.

```java
// Example within a system that processes an explosion event
void applyExplosionKnockback(Entity target, Vector3d impulse) {
    // Avoid stacking multiple knockback components; check for an existing one first.
    if (target.hasComponent(KnockbackComponent.class)) {
        return;
    }

    KnockbackComponent knockback = new KnockbackComponent();
    knockback.setVelocity(impulse);
    knockback.setVelocityType(ChangeVelocityType.Add);
    knockback.setDuration(0.4f); // Knockback lasts for 0.4 seconds

    // A separate system might add a modifier based on the entity's armor
    // knockback.addModifier(0.75); 

    target.addComponent(knockback);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Timer Manipulation:** Do not call `incrementTimer` or `setTimer` from game logic systems. The component's lifecycle is managed exclusively by the core entity update loop. Manual manipulation will break the effect's duration.
- **Component Re-use:** Do not detach a KnockbackComponent from one entity and attach it to another. The internal state (like the timer) makes it unsafe for re-use. Always create a new instance for each new knockback event.
- **Asynchronous Modification:** Never modify a component's state from a separate thread. All game logic that interacts with components must be executed on the main server thread.

## Data Pipeline
The KnockbackComponent serves as a critical data packet that flows from game logic to the physics engine.

> Flow:
> Game Event (e.g., Arrow Hit) -> CombatSystem -> **KnockbackComponent (Created & Attached)** -> EntityUpdateSystem -> Physics State Update -> Network Sync (Client receives new velocity)

