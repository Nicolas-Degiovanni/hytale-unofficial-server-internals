---
description: Architectural reference for ForceAccumulator
---

# ForceAccumulator

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient State Object

## Definition
```java
// Signature
public class ForceAccumulator {
```

## Architecture & Concepts

The ForceAccumulator is a short-lived, stateful object that serves a critical role within the server-side physics simulation loop. It is not a persistent system but rather a transient data structure used to aggregate the various forces acting upon a single PhysicsBody during a single simulation tick.

Its core design purpose is to implement the **Accumulator Pattern**. This pattern decouples the main physics integration algorithm from the specific sources of force (e.g., gravity, player input, friction, fluid dynamics). The primary physics system instantiates a ForceAccumulator, passes it to a collection of ForceProvider implementations, and each provider contributes to the total force by modifying the accumulator's public *force* vector.

This architecture makes the physics system highly extensible. New physical behaviors can be introduced by simply creating a new ForceProvider implementation without modifying the core, performance-sensitive integration logic.

## Lifecycle & Ownership

The lifecycle of a ForceAccumulator is extremely brief and tightly controlled by the physics engine. Understanding this is critical to prevent state corruption.

-   **Creation:** A new ForceAccumulator is instantiated by the primary physics integration system at the beginning of a physics update for a single PhysicsBody. It is typically created on the stack within the main tick method.
-   **Scope:** The object's lifetime is strictly confined to the duration of one physics integration step for one entity. It exists only to calculate the net force for that specific frame.
-   **Destruction:** It becomes eligible for garbage collection immediately after the `computeResultingForce` method completes and its resulting *force* vector has been consumed by the physics integrator to calculate acceleration.

**WARNING:** Instances of this class are not designed to be cached, reused across ticks, or shared between entities. Doing so will lead to severe and difficult-to-diagnose physics simulation errors.

## Internal State & Concurrency

-   **State:** The ForceAccumulator is fundamentally a mutable object. Its entire purpose is to have its internal state—primarily the *force* vector—modified by external systems (ForceProviders). The state is reset to a clean slate at the beginning of every calculation by the internal `initialize` call.

-   **Thread Safety:** **This class is not thread-safe and must be treated as thread-local.** It is designed for synchronous, single-threaded execution within the context of a single entity's physics tick. Any attempt to share an instance across threads or allow multiple threads to interact with the same instance will result in race conditions and non-deterministic physics calculations.

## API Surface

The public contract of this class is primarily its mutable fields, which are accessed by ForceProvider implementations. The methods orchestrate the calculation but are not intended for general use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| force | Vector3d | O(1) | **Primary Output.** The net force vector. This field is mutated by each ForceProvider to accumulate the total force. |
| speed | double | O(1) | **Read-Only Input.** The scalar magnitude of the body's velocity at the start of the tick. Used by providers to calculate velocity-dependent forces like drag. |
| resistanceForceLimit | Vector3d | O(1) | **Read-Only Input.** The force vector required to bring the body to a complete stop in one time step. Used by providers to calculate friction. |
| computeResultingForce(...) | protected void | O(N) | **Entry Point.** Orchestrates the entire calculation. Resets state and iterates through N ForceProviders. Intended for use only by the core physics package. |

## Integration Patterns

### Standard Usage

The ForceAccumulator is not used directly by most game logic. Instead, it is managed by a higher-level physics system. A developer's primary interaction is with the ForceProvider interface, which receives an accumulator instance.

```java
// Hypothetical usage within a core PhysicsSystem class

// 1. A new accumulator is created for each body, each tick.
ForceAccumulator accumulator = new ForceAccumulator();
PhysicsBodyState currentState = targetBody.getState();
ForceProvider[] activeProviders = getActiveForceProvidersFor(targetBody);

// 2. The protected method is called to run the simulation step.
accumulator.computeResultingForce(
    currentState,
    targetBody.isOnGround(),
    activeProviders,
    targetBody.getMass(),
    timeStep
);

// 3. The resulting force is extracted and used for physics integration.
Vector3d netForce = accumulator.force;
Vector3d acceleration = netForce.scale(1.0 / targetBody.getMass());
targetBody.applyAcceleration(acceleration, timeStep);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not store and reuse a ForceAccumulator instance across multiple ticks or for different entities. Its internal state is only valid for the single computation in which it was created.
-   **Pre-emptive State Modification:** Do not modify the *force* vector before passing the accumulator to the `computeResultingForce` method. The internal `initialize` call will overwrite any changes.
-   **Concurrent Access:** Never pass a ForceAccumulator instance to another thread. Each physics thread must manage its own set of accumulators.

## Data Pipeline

The ForceAccumulator acts as a temporary aggregation point for data flowing from various game systems into the final physics integration step.

> Flow:
> Physics Tick Start -> **ForceAccumulator** (Creation & Initialization) -> Collection of ForceProviders (Read Body State, Write to `force` vector) -> **ForceAccumulator** (Final `force` vector is complete) -> Physics Integrator (Reads `force` vector) -> Entity Velocity & Position Update

