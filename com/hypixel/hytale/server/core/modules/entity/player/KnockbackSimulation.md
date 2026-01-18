---
description: Architectural reference for KnockbackSimulation
---

# KnockbackSimulation

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Component (Data Object)

## Definition
```java
// Signature
public class KnockbackSimulation implements Component<EntityStore> {
```

## Architecture & Concepts
The KnockbackSimulation component is a server-side, ephemeral state container used to manage the physics and state of a player entity during a server-authoritative knockback event. It is a core element of the server's movement reconciliation and anti-cheat strategy.

In Hytale's Entity-Component-System (ECS) architecture, this component is attached to a player entity to temporarily override its normal movement logic. Instead of relying purely on client input for movement, the server executes a short, predictable physics simulation. This component holds all the necessary state for this simulation, including the initial force, the simulated position and velocity, and the remaining duration.

Its primary function is to act as a bridge between an external event (like an explosion or weapon hit) and the continuous player movement processing loop. It ensures that external forces are applied reliably and cannot be ignored by a modified client, while also attempting to blend the server's simulation with the client's own predicted movement to maintain a smooth player experience.

### Lifecycle & Ownership
The lifecycle of a KnockbackSimulation component is intentionally brief and tied directly to the duration of the knockback effect.

-   **Creation:** This component is never instantiated directly. It is attached to a player entity by a high-level game system (e.g., a combat or physics system) in response to a game event that should cause knockback. The component is added to the entity via the `EntityStore`.
-   **Scope:** The component's lifetime is dictated by its internal `remainingTime` field, which is initialized to `KNOCKBACK_SIMULATION_TIME` (0.5 seconds). It persists only for this duration while being actively processed by the server's movement systems.
-   **Destruction:** The component is removed from the entity by the responsible game system (typically the `PlayerMovementSystem`) once `remainingTime` decrements to zero or less. After removal, the entity's movement logic reverts to its standard, client-driven behavior.

## Internal State & Concurrency
-   **State:** This component is highly mutable and serves as a transient data cache. It aggregates client-reported state (clientPosition, clientMovementStates) and server-calculated state (simPosition, simVelocity) for the duration of the simulation. All fields are considered volatile and are updated on nearly every server tick that the component is active.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be owned and manipulated exclusively by the single server thread responsible for ticking the world and the entities within it. Any concurrent access from network threads, AI threads, or other systems will result in state corruption, desynchronization, and undefined behavior.

    **WARNING:** All interactions with this component, including its creation and modification, must be marshaled to the entity's owner game loop thread.

## API Surface
The public API is primarily composed of state accessors and mutators used by the server's core movement and combat systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type definition for this component, used by the ECS framework. |
| addRequestedVelocity(Vector3d) | void | O(1) | Adds a velocity vector to the pending knockback force. Used to stack multiple knockback sources. |
| setRequestedVelocity(Vector3d) | void | O(1) | Sets the knockback velocity, overwriting any previously requested force. Used to apply a single, definitive knockback. |
| reset() | void | O(1) | Resets the simulation's internal timer to its default start time (0.5s). |
| consumeWasJumping() | boolean | O(1) | A state consumption method. Returns if the player was jumping and immediately resets the internal flag to false. |

## Integration Patterns

### Standard Usage
The component is always managed through an entity. A system initiates the knockback, and the `PlayerMovementSystem` consumes the data each tick to execute the simulation.

```java
// In a CombatSystem, after a hit is confirmed:
Entity playerEntity = getPlayerEntity();

// Add the component if it doesn't exist, or get the existing one.
KnockbackSimulation sim = playerEntity.getOrAddComponent(KnockbackSimulation.class);

// Reset the timer and apply the force.
sim.reset();
sim.setRequestedVelocity(knockbackVector);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new KnockbackSimulation()`. The resulting object will not be managed by the ECS framework and will be completely ignored by all game systems. This is a "dead" component.
-   **State Caching:** Do not fetch a reference to this component and store it in a local variable across multiple ticks. The component is ephemeral and can be removed at any time, which will lead to NullPointerExceptions or processing of stale data. Always retrieve it from the entity on the tick you need it.
-   **Direct State Manipulation:** Avoid directly setting fields like `simPosition` or `remainingTime` from outside the core `PlayerMovementSystem`. These values are tightly controlled by the simulation loop. Use methods like `setRequestedVelocity` to influence the simulation as intended.

## Data Pipeline
The flow of data for a knockback event is unidirectional, starting from a game event and flowing through this component into the core physics simulation.

> Flow:
> Game Event (e.g., Explosion) -> Combat System -> **KnockbackSimulation** component attached to Entity -> Player Movement System (reads component state each tick) -> Updates Entity Transform -> Server-to-Client Position Update Packet<ctrl63>

