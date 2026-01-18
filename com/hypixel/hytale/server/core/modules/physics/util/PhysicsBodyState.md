---
description: Architectural reference for PhysicsBodyState
---

# PhysicsBodyState

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient Data Object

## Definition
```java
// Signature
public class PhysicsBodyState {
```

## Architecture & Concepts
The PhysicsBodyState class is a fundamental data structure, not a service or manager. It serves as a Plain Old Java Object (POJO) designed to encapsulate the physical state of an object at a single, discrete point in time. Its primary role is to act as a data carrier, transporting position and velocity information between different stages of the server-side physics simulation pipeline.

This class is intentionally minimal. It holds no logic and performs no calculations. Systems like the core PhysicsSystem or collision detection modules operate *on* PhysicsBodyState instances, using them as input for calculations or as output to represent a new, predicted state. It is a foundational component for implementing physics features such as interpolation, extrapolation, and state replication over the network.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically within the main physics simulation loop. For example, a physics integration step might instantiate a new PhysicsBodyState to store the result of its calculations for a given entity in the current tick.
- **Scope:** The lifetime of a PhysicsBodyState object is expected to be extremely short, often confined to the scope of a single method or a single simulation tick. They are ephemeral by design.
- **Destruction:** Objects are managed by the Java Garbage Collector. As they are intended for short-term use, they become eligible for garbage collection as soon as all references within the tick's calculation scope are dropped. There is no manual destruction or pooling mechanism.

**WARNING:** Holding references to PhysicsBodyState instances beyond a single simulation tick is a severe anti-pattern that can lead to memory leaks and the use of stale, incorrect physics data.

## Internal State & Concurrency
- **State:** The state is **Mutable**. While the references to the Vector3d objects for position and velocity are declared as final and cannot be reassigned, the underlying Vector3d objects themselves are fully mutable. Any system with a reference to a PhysicsBodyState instance can directly modify the x, y, and z components of its position and velocity vectors.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. Concurrent modification of the position or velocity vectors from multiple threads will lead to race conditions, data corruption, and non-deterministic physics behavior.

**WARNING:** Access to a PhysicsBodyState instance must be strictly confined to a single thread, typically the main server physics thread. If state must be passed between threads, a deep copy of the instance must be created to ensure thread isolation.

## API Surface
The public contract of this class consists of direct field access. There are no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| position | Vector3d | O(1) | The absolute world-space position of the body. |
| velocity | Vector3d | O(1) | The linear velocity of the body, in units per second. |

## Integration Patterns

### Standard Usage
This class is used as a data container to pass state between physics subsystems or to store the result of a calculation.

```java
// Example: Calculating the next state in a physics update
public PhysicsBodyState predictNextState(PhysicsBodyState currentState, double deltaTime) {
    PhysicsBodyState nextState = new PhysicsBodyState();

    // Copy current position
    nextState.position.set(currentState.position);

    // Integrate velocity to find next position
    nextState.position.add(currentState.velocity.x * deltaTime, currentState.velocity.y * deltaTime, currentState.velocity.z * deltaTime);

    // Assume velocity is constant for this example
    nextState.velocity.set(currentState.velocity);

    return nextState;
}
```

### Anti-Patterns (Do NOT do this)
- **State Sharing:** Do not pass the same PhysicsBodyState instance to multiple, independent simulation systems that may modify it. This creates implicit coupling and unpredictable behavior. Always pass copies for mutation.
- **Concurrent Modification:** Never modify a PhysicsBodyState from one thread while another thread is reading from it. This is a classic race condition.
- **Long-Term Storage:** Do not store PhysicsBodyState instances in long-lived collections. They represent a point-in-time snapshot and should be discarded after use.

## Data Pipeline
PhysicsBodyState acts as a data record that flows through the physics engine. It does not process data itself but is the data being processed.

> Flow:
> PhysicsSystem (Read Current State) -> **PhysicsBodyState (New Instance)** -> Physics Integrator (Populate Instance) -> Collision Detector (Read Instance) -> State Finalizer (Commit to Entity)

