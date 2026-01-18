---
description: Architectural reference for PhysicsBodyStateUpdaterSymplecticEuler
---

# PhysicsBodyStateUpdaterSymplecticEuler

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility

## Definition
```java
// Signature
public class PhysicsBodyStateUpdaterSymplecticEuler extends PhysicsBodyStateUpdater {
```

## Architecture & Concepts
The PhysicsBodyStateUpdaterSymplecticEuler is a concrete implementation of the **Strategy Pattern** for physics integration. It is responsible for advancing the state of a single physics body (position, velocity) over a discrete time step, a process often called "solving" or "integrating".

Its core architectural purpose is to decouple the numerical integration algorithm from the main physics simulation loop. The server's physics engine can select this specific updater based on configuration or performance requirements, without altering the high-level code that manages the collection of physics bodies.

The key characteristic of this class is its use of the **Symplectic Euler** method. Unlike a standard Euler integrator which uses the velocity from the *start* of the time step to calculate the new position, the Symplectic method first calculates the new velocity for the *end* of the time step, and then uses that new velocity to update the position.

`v_new = v_old + acceleration * dt`
`p_new = p_old + v_new * dt`

This approach offers significantly better energy conservation and stability in long-running simulations, preventing the "exploding" artifacts common with simpler Euler methods. It is a common and robust choice for real-time game physics.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's `PhysicsModule` or a similar bootstrap manager during server initialization. It is a system-level object representing a specific, stateless algorithm.
-   **Scope:** This object is a singleton. A single instance persists for the entire lifetime of the server process and is shared across all physics simulations.
-   **Destruction:** The object is destroyed and eligible for garbage collection only upon server shutdown.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no member fields and its behavior depends exclusively on the arguments passed to its `update` method. Its primary function is to mutate the `after` PhysicsBodyState object passed into it.
-   **Thread Safety:** The class is inherently **thread-safe**. As a stateless utility, a single instance can be safely invoked by multiple threads simultaneously, provided each thread is operating on distinct `PhysicsBodyState` instances. This design is crucial for a potentially multi-threaded server physics engine.

## API Surface
The public contract consists of a single method inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(before, after, mass, dt, onGround, forceProvider) | void | O(N) | Advances a physics body's state over a single time step `dt`. Complexity is linear to N, the number of `ForceProvider` instances. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked by the core physics simulation loop, which iterates over all active physics bodies and applies the configured updater.

```java
// Conceptual example from within a PhysicsWorld update loop
PhysicsBodyStateUpdater integrator = physicsContext.getIntegrator(); // Returns the configured singleton

for (PhysicsBody body : allBodiesInWorld) {
    PhysicsBodyState currentState = body.getCurrentState();
    PhysicsBodyState nextState = body.getNextState(); // A pre-allocated state object
    
    // The integrator calculates the next state based on the current one
    integrator.update(currentState, nextState, body.getMass(), deltaTime, body.isOnGround(), body.getForceProviders());

    // The states are then swapped for the next frame
    body.swapStates();
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new PhysicsBodyStateUpdaterSymplecticEuler()`. The physics engine is responsible for providing the globally shared instance. Direct instantiation bypasses engine configuration and creates unnecessary object churn.
-   **State Aliasing:** Passing the same object instance for both the `before` and `after` parameters will result in catastrophic simulation errors. The calculation relies on reading from an unmodified `before` state while writing to a distinct `after` state.
-   **Variable Time Steps:** While the method accepts a `dt` parameter, the Symplectic Euler method is most stable with a small, fixed time step. Feeding it large or highly variable `dt` values from the main game loop can introduce significant simulation instability and error.

## Data Pipeline
This component acts as a pure function that transforms an initial state into a final state based on external forces and a time delta.

> Flow:
> Array of ForceProvider -> **PhysicsBodyStateUpdaterSymplecticEuler**.update() -> Mutated `after` PhysicsBodyState

