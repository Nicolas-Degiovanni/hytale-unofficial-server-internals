---
description: Architectural reference for PhysicsBodyStateUpdaterMidpoint
---

# PhysicsBodyStateUpdaterMidpoint

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient Utility / Strategy

## Definition
```java
// Signature
public class PhysicsBodyStateUpdaterMidpoint extends PhysicsBodyStateUpdater {
```

## Architecture & Concepts
The PhysicsBodyStateUpdaterMidpoint is a concrete implementation of the **Strategy** pattern for physics integration. It encapsulates a specific numerical integration algorithm—the Midpoint method, also known as the second-order Runge-Kutta method (RK2).

This class is a core component of the server-side physics simulation loop. Its primary responsibility is to accurately advance the state of a physical body (position, velocity) from one point in time to the next, given a set of applied forces.

Unlike a simpler Euler integrator, which calculates acceleration only at the beginning of a time step, the Midpoint method provides superior accuracy and stability by performing a two-stage calculation:
1.  It first estimates the body's state at the *midpoint* of the time step.
2.  It then uses the forces and velocity at this midpoint to compute a more accurate final state for the end of the time step.

This approach significantly reduces integration errors, making it suitable for simulations where forces change rapidly or where higher fidelity is required to prevent instability. It represents a trade-off, offering better physical accuracy at the cost of increased computation compared to its base-class counterparts.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level physics system or module, such as a PhysicsWorld or SimulationLoop, during its own initialization phase. It is not intended to be managed as a global singleton.
-   **Scope:** The lifetime of a PhysicsBodyStateUpdaterMidpoint instance is typically tied to its owning physics simulation context. It persists as long as the simulation is active.
-   **Destruction:** The object is eligible for garbage collection when the owning physics system is shut down or reconfigured to use a different integration strategy.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no member fields and all required data is provided via method arguments. The outcome of the `update` method depends solely on its inputs.
-   **Thread Safety:** As a stateless object, PhysicsBodyStateUpdaterMidpoint is inherently **thread-safe**. A single instance can be safely shared and used by multiple worker threads in a parallelized physics engine to update different physics bodies concurrently without locks or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(before, after, mass, dt, onGround, forceProvider) | void | O(N) | Advances the physics state using the Midpoint method. Complexity is linear to N, the number of ForceProviders. Mutates the `after` state object. |

## Integration Patterns

### Standard Usage
This class is designed to be used by a central physics simulation loop. The loop iterates over all dynamic bodies, providing their current state and a destination state object to this updater.

```java
// Within a server-side physics simulation loop
PhysicsBodyState currentState = body.getCurrentState();
PhysicsBodyState nextState = body.getNextState(); // A separate object to store results
ForceProvider[] forces = world.getForcesActingOn(body);

// The updater is typically a member of the simulation class
this.midpointUpdater.update(currentState, nextState, body.getMass(), FIXED_DELTA_TIME, body.isOnGround(), forces);

// The nextState is now committed
body.commitState(nextState);
```

### Anti-Patterns (Do NOT do this)
-   **State Object Aliasing:** Never pass the same object instance for both the `before` and `after` parameters. Doing so will corrupt the calculation as intermediate results will overwrite the initial state, leading to unpredictable and incorrect simulation behavior.
-   **Variable Time Steps:** While the Midpoint method is more stable than Euler integration, feeding it a variable or large time delta (`dt`) can still lead to simulation instability, object tunneling, or other artifacts. The physics engine should always call this method with a small, fixed time step.

## Data Pipeline
The class acts as a pure computational function within the physics engine's data transformation pipeline for a single simulation tick.

> Flow:
> Previous Tick's PhysicsBodyState -> Active ForceProvider Array -> **PhysicsBodyStateUpdaterMidpoint** -> Mutated `after` PhysicsBodyState -> Physics World Commit for Next Tick

