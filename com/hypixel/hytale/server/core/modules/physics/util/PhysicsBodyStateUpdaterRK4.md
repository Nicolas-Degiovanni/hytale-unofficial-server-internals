---
description: Architectural reference for PhysicsBodyStateUpdaterRK4
---

# PhysicsBodyStateUpdaterRK4

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility

## Definition
```java
// Signature
public class PhysicsBodyStateUpdaterRK4 extends PhysicsBodyStateUpdater {
```

## Architecture & Concepts
The PhysicsBodyStateUpdaterRK4 is a concrete implementation of the physics integration strategy. Its sole responsibility is to advance the simulation state of a single physical body from time *t* to *t + dt* using the **Runge-Kutta 4th order (RK4) numerical integration method**.

This class represents a high-precision approach to physics simulation. Unlike simpler Euler integration, which evaluates forces only once at the beginning of the time step, the RK4 method performs four evaluations at different points within the time step. This significantly reduces integration error, resulting in a more stable and accurate simulation, especially for objects under complex or rapidly changing forces (e.g., orbits, springs, complex aerodynamics).

It is a core computational component of the server-side physics engine. It is intentionally decoupled from higher-level concepts like entities or game rules, operating purely on the mathematical constructs of state, mass, and force. Its use implies a design choice favoring simulation accuracy over raw computational performance, as it is more expensive than lower-order methods.

### Lifecycle & Ownership
- **Creation:** Instantiated by a high-level physics system, such as a PhysicsWorld or SimulationManager, during its initialization. It is not intended for direct creation by game logic developers.
- **Scope:** The object's lifetime is typically bound to its parent physics system. To prevent frequent memory allocation, a single instance is often created and reused for the entire server session.
- **Destruction:** The object is eligible for garbage collection when the owning physics system is shut down or reconfigured to use a different integration strategy.

## Internal State & Concurrency
- **State:** This class is stateful *during the execution of the update method*. It contains a private, mutable PhysicsBodyState field named *state*. This field is used as a temporary scratchpad to store intermediate calculations required by the multi-step RK4 algorithm. This is a critical performance optimization that avoids heap allocations within the tight physics loop. The data in this field is transient and holds no meaning between calls to the update method.
- **Thread Safety:** **This class is not thread-safe.** The use of a reusable internal state object makes it inherently unsafe for concurrent use. If a single instance is accessed by multiple threads simultaneously, the intermediate calculations will be corrupted, leading to unpredictable and catastrophic physics simulation failures.

**WARNING:** Each physics simulation thread must have its own unique instance of PhysicsBodyStateUpdaterRK4. Do not share instances across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(before, after, mass, dt, onGround, forceProvider) | void | O(k) | Integrates the physics state forward by time *dt*. Mutates the *after* parameter with the result. *k* is the number of ForceProviders. |

## Integration Patterns

### Standard Usage
This class is an internal component of the physics engine. It is invoked by a managing system that iterates over all simulated bodies during a physics tick. The manager is responsible for gathering forces and passing the correct state objects.

```java
// Hypothetical usage within a PhysicsWorld simulation loop
PhysicsBodyStateUpdater integrator = new PhysicsBodyStateUpdaterRK4(); // Created once during initialization

// Inside the physics tick method...
for (PhysicsBody body : allSimulatedBodies) {
    ForceProvider[] forces = forceAccumulator.getForcesFor(body);
    PhysicsBodyState currentState = body.getCurrentState();
    PhysicsBodyState nextState = body.getNextStateWriteBuffer();

    // Execute the RK4 integration
    integrator.update(currentState, nextState, body.getMass(), deltaTime, body.isOnGround(), forces);

    body.commitNextState(); // Swaps the write buffer to be the new current state
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Game logic should never create or manage instances of this class. The choice of integrator is an engine-level configuration.
- **Concurrent Access:** Never share a single instance of this class across multiple threads. The internal state used for calculations will become corrupted.
- **State Reuse:** Do not attempt to read the internal *state* field of this class. It is an implementation detail and its contents are only valid during a single *update* call.

## Data Pipeline
This component acts as a transformation function within the physics loop, converting state at time *t* to state at time *t + dt*.

> **Flow:**
>
> 1.  **Input:** The physics engine provides the current **PhysicsBodyState** (position, velocity), entity **mass**, a **delta time**, and an array of active **ForceProvider** objects (e.g., gravity, player thrust).
> 2.  **Processing (Internal):** The `update` method executes the four stages of the RK4 algorithm:
>     -   k1: Evaluate forces at the start of the interval.
>     -   k2: Evaluate forces at the midpoint of the interval, using k1.
>     -   k3: Evaluate forces again at the midpoint, using k2.
>     -   k4: Evaluate forces at the end of the interval, using k3.
> 3.  **Integration:** The four evaluations are combined in a weighted average to compute the final change in velocity and position.
> 4.  **Output:** The **`after` PhysicsBodyState** parameter, passed by reference, is mutated to contain the newly calculated position and velocity for the next simulation tick.

