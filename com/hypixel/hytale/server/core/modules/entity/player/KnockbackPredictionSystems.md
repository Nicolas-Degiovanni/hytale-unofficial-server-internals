---
description: Architectural reference for KnockbackPredictionSystems
---

# KnockbackPredictionSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class KnockbackPredictionSystems {
    // Contains multiple nested static System classes
}
```

## Architecture & Concepts
The **KnockbackPredictionSystems** class is not an instantiable object but rather a static container and namespace for a suite of related Entity Component Systems (ECS). These systems collectively manage the server-side prediction of player movement during a knockback event.

The primary goal of this module is to create a smooth visual experience for the client while maintaining server authority. When a player is subjected to knockback, the server initiates a short-term, isolated physics simulation. This simulation takes into account both the initial knockback force and continuous input from the client (e.g., air-strafing). By simulating the player's trajectory on the server in a way that closely mirrors the client's own prediction, the system avoids the jarring corrections and rubber-banding that would otherwise occur.

This entire mechanism is orchestrated through the **KnockbackSimulation** component. The presence of this component on a player entity activates these systems. Its removal signifies the end of the predictive simulation, at which point the player's state is finalized.

The module is composed of the following key systems, each with a distinct responsibility:
*   **InitKnockback**: Initializes the simulation state when it begins.
*   **CaptureKnockbackInput**: Ingests movement updates from the client during the simulation.
*   **SimulateKnockback**: The core physics loop that ticks the simulation forward.
*   **ClearOnTeleport** & **ClearOnRemove**: Housekeeping systems that terminate the simulation under specific conditions.

---
description: Architectural reference for KnockbackPredictionSystems.SimulateKnockback
---

# KnockbackPredictionSystems.SimulateKnockback

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** ECS System

## Definition
```java
// Signature
@Deprecated
public static class SimulateKnockback extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
**SimulateKnockback** is the computational core of the knockback prediction module. As an **EntityTickingSystem**, it is executed by the ECS scheduler on every server tick for all entities that possess the required components defined by its query (primarily **KnockbackSimulation** and **TransformComponent**).

This system runs a fixed-timestep physics simulation that is decoupled from the server's main tick rate. It models the trajectory of a player entity under the influence of an initial velocity impulse, gravity, air drag, and ongoing client movement inputs. Its purpose is to predict where the client *will be* and gently guide the authoritative server-side entity towards that predicted state.

The simulation is stateful, but the state is stored entirely within the **KnockbackSimulation** component, not the system itself. This allows the system to remain transient and thread-agnostic, as the ECS framework guarantees safe access to components during the tick execution.

**WARNING:** This class is marked as **Deprecated**. This indicates it may be part of a legacy physics implementation and could be superseded by more modern or unified physics systems in the engine. Use with caution and expect potential replacement.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS scheduler for the duration of the `tick` method execution. It is a short-lived, throwaway object.
-   **Scope:** The system's logic is scoped to entities matching its component query. The simulation *itself* is scoped to the lifetime of the **KnockbackSimulation** component on a given entity.
-   **Destruction:** The system instance is eligible for garbage collection immediately after the `tick` method completes. The simulation is terminated when the system removes the **KnockbackSimulation** component, either because its timer has expired or because the entity has died.

## Internal State & Concurrency
-   **State:** This system is stateless. All simulation variables, such as current velocity, position, and remaining time, are stored in the **KnockbackSimulation** component attached to the entity.
-   **Thread Safety:** This system is not thread-safe and must only be invoked by the main server thread via the ECS scheduler. It performs mutable operations on components and interacts with non-thread-safe world resources like the **CollisionModule**.

## API Surface
The public contract is exclusively with the ECS framework through the overridden `tick` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | Executes one frame of the simulation. N is the number of fixed-timestep physics steps required to cover the delta time `dt`. Throws assertion errors if required components are missing. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Integration is achieved by adding a **KnockbackSimulation** component to a player entity. The ECS framework will automatically discover the entity and schedule this system to process it.

```java
// To initiate a knockback simulation on a player entity (ref):
KnockbackSimulation sim = new KnockbackSimulation(duration, initialVelocity);
commandBuffer.addComponent(ref, sim);

// The SimulateKnockback system will now run on this entity automatically.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new SimulateKnockback()`. The ECS framework manages its lifecycle.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the dependency ordering and thread safety guarantees of the ECS scheduler, leading to race conditions and unpredictable behavior.
-   **State Management:** Do not attempt to read the simulation's state mid-tick from another thread. All state should be considered volatile until the full ECS tick is complete.

## Data Pipeline
This system processes data from various components to update the entity's physical state.

> Flow:
> **KnockbackSimulation** (Input Velocity, Time) + **PlayerInput** (Client Movement) -> **SimulateKnockback Physics Loop** (Gravity, Drag, Collision) -> **KnockbackSimulation** (Updated State) -> **TransformComponent** (Updated Server Position)

---
description: Architectural reference for KnockbackPredictionSystems.CaptureKnockbackInput
---

# KnockbackPredictionSystems.CaptureKnockbackInput

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** ECS System

## Definition
```java
// Signature
public static class CaptureKnockbackInput extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This system acts as the primary data ingestion point for client actions during a knockback simulation. Its sole responsibility is to poll the **PlayerInput** component for new movement updates and transfer them into the **KnockbackSimulation** component.

It is critically configured to run *before* the main **PlayerSystems.ProcessPlayerInput** system. This ordering ensures that knockback-related inputs are captured and handled by the specialized simulation logic first, preventing the standard movement systems from interfering with the knockback trajectory.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS scheduler for the duration of the `tick` method execution.
-   **Scope:** Operates on any entity that has both a **PlayerInput** and a **KnockbackSimulation** component.
-   **Destruction:** The instance is discarded after the `tick` method completes.

## Integration Patterns

### Standard Usage
This system is part of the internal knockback module and requires no direct developer interaction. It is automatically active for any player undergoing a knockback simulation.

## Data Pipeline
The data flow is simple and unidirectional, acting as a bridge between two components.

> Flow:
> **PlayerInput** component (MovementUpdateQueue) -> **CaptureKnockbackInput** -> **KnockbackSimulation** component (ClientPosition, RelativeMovement)

