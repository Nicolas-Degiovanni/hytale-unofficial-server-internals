---
description: Architectural reference for PhysicsBodyStateUpdater
---

# PhysicsBodyStateUpdater

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient

## Definition
```java
// Signature
public class PhysicsBodyStateUpdater {
```

## Architecture & Concepts
The PhysicsBodyStateUpdater is a fundamental component of the server-side physics engine, responsible for implementing the core integration step. Its primary function is to advance the state of a physical body—specifically its position and velocity—from one discrete time step to the next.

This class acts as a stateful calculator. It is designed to be instantiated, used for a single physics tick calculation for a single entity, and then discarded. It encapsulates the mathematical formulas for numerical integration, specifically a semi-implicit Euler method where position is updated using the *old* velocity, and then velocity is updated using the newly computed acceleration.

It employs a Strategy pattern for force calculation via the ForceProvider array. The updater does not need to know the origin of forces (e.g., gravity, player input, explosions); it only consumes them through the ForceProvider interface, sums them using its internal ForceAccumulator, and computes a final acceleration based on Newton's second law (a = F/m).

## Lifecycle & Ownership
- **Creation:** Instantiated directly via `new PhysicsBodyStateUpdater()`. It is expected to be created by a higher-level system that manages the physics simulation loop, likely on a per-entity, per-tick basis.
- **Scope:** Extremely short-lived. An instance's lifetime should be confined to a single call of the physics simulation for one entity. Its internal state is only valid for the duration of the `update` method call.
- **Destruction:** The object is small and has no external resource handles. It is intended to be garbage collected immediately after its use in a single physics tick.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains an internal `acceleration` vector and a `ForceAccumulator` instance. These fields are mutated during every call to the `update` method. This design avoids heap allocations for these vector objects on every physics tick, optimizing for performance.

- **Thread Safety:** **CRITICAL:** This class is not thread-safe. Its mutable internal state makes it fundamentally unsafe for concurrent access. Sharing a single instance across multiple threads will lead to race conditions and unpredictable physics calculations. Each thread processing physics simulations must use its own distinct instance of PhysicsBodyStateUpdater.

## API Surface
The public contract is intentionally minimal, exposing only the primary `update` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(before, after, mass, dt, onGround, forceProviders) | void | O(N) | Executes a full physics integration step. N is the number of ForceProviders. Mutates the `after` state object. This is the sole entry point for the class. |

## Integration Patterns

### Standard Usage
The updater is designed to be used within a physics simulation loop. For each entity being updated, a new or recycled updater instance is used to compute the next state.

```java
// Within a server-side physics simulation loop for a single entity
PhysicsBodyState currentState = entity.getPhysicsState();
PhysicsBodyState nextState = new PhysicsBodyState();
ForceProvider[] forces = { new GravityForce(), new PlayerInputForce() };

// A new updater is used for this specific calculation
PhysicsBodyStateUpdater updater = new PhysicsBodyStateUpdater();
updater.update(currentState, nextState, entity.getMass(), deltaTime, entity.isOnGround(), forces);

// The entity's state is now advanced for the next tick
entity.setPhysicsState(nextState);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache and reuse a single instance of PhysicsBodyStateUpdater across multiple entities within the same physics tick. The internal state from one entity's calculation will corrupt the next.
- **Concurrent Access:** **WARNING:** Never share an instance between threads. If the physics engine is multi-threaded, each worker thread must create its own instances of this class.

## Data Pipeline
The flow of data through this component is linear and unidirectional for a single tick.

> Flow:
> Array of ForceProvider -> ForceAccumulator -> **PhysicsBodyStateUpdater** -> Acceleration Calculation -> Integration -> Mutated `after` PhysicsBodyState

