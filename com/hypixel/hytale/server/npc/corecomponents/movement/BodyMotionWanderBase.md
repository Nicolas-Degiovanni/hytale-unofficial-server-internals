---
description: Architectural reference for BodyMotionWanderBase
---

# BodyMotionWanderBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Component Logic

## Definition
```java
// Signature
public abstract class BodyMotionWanderBase extends BodyMotionBase {
```

## Architecture & Concepts

BodyMotionWanderBase is an abstract base class that provides the core logic for non-goal-oriented, exploratory movement for Non-Player Characters (NPCs). It operates as a state machine, enabling an entity to intelligently wander within an environment without relying on a global navigation mesh or predefined paths. This component is fundamental for ambient behaviors, giving life to creatures that patrol, graze, or simply meander.

The central design pattern is a **probe-based environmental query system**. Instead of performing a computationally expensive pathfinding search (like A*), the component tests a discrete number of directions around the NPC. For each direction, it "probes" the world using the NPC's `MotionController` to determine if a path is viable and how far the NPC can travel before encountering an obstacle. This makes it highly efficient for localized, dynamic movement where a long-term destination is not required.

The component's behavior is governed by a simple but effective state machine with four states:
1.  **SEARCHING:** The NPC is stationary, actively probing the environment in a cone ahead of it to find the best possible direction for its next movement segment.
2.  **TURNING:** A valid direction has been chosen. The NPC rotates on the spot to face the new heading before moving forward. This ensures smooth and believable motion.
3.  **WALKING:** The NPC moves towards the chosen target position for a randomized duration. It uses a `SteeringForcePursue` behavior to smoothly arrive at the destination.
4.  **STOPPED:** The NPC is inactive and will not attempt to move.

This class is abstract, requiring concrete implementations to provide environment-specific constraints via the `constrainMove` method. This allows for flexible reuse, where subclasses can define movement boundaries such as zone borders, water avoidance, or altitude limits.

### Lifecycle & Ownership
-   **Creation:** Instantiated via its corresponding `BuilderBodyMotionWanderBase` during the NPC asset loading and definition phase. It is never created directly with `new`. This component is part of an NPC's `Role` definition.
-   **Scope:** The component's lifecycle is tied to the NPC's active `Role`. It persists as long as the NPC is assigned a role that includes this wandering behavior. The `activate` method is called when the behavior becomes active, and `deactivate` is called when it ceases.
-   **Destruction:** The object is marked for garbage collection when the parent NPC entity is destroyed or its `Role` is changed to one that does not utilize this component.

## Internal State & Concurrency
-   **State:** This component is highly stateful and mutable. It maintains the current state of its internal state machine (e.g., `SEARCHING`, `WALKING`), timers for movement duration (`walkTime`), the calculated target position (`targetPosition`), and cached results from its environmental probes (`walkDistances`). This state is not intended to be modified externally.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively owned and operated by the server's main game loop thread that ticks the NPC's AI. All method calls and state modifications must occur on this thread to prevent race conditions and data corruption.

## API Surface

The public API is minimal, designed for interaction with the parent `Role` system rather than general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(...) | void | O(1) | Initializes the state machine by triggering a new search. Called when the component becomes the active motion driver. |
| deactivate(...) | void | O(1) | Performs minor cleanup, primarily for debug visualizations. |
| computeSteering(...) | boolean | O(N) | The primary update method. Executes one tick of the state machine logic. N is bounded by `testsPerTick`. |
| motionControllerChanged(...) | void | O(1) | Resets the state machine. Critical for adapting when the NPC's physical movement capabilities change (e.g., switching from walking to flying). |
| constrainMove(...) | abstract double | Varies | Abstract contract for subclasses to implement boundary checks or other movement constraints. |

## Integration Patterns

### Standard Usage

This component is not used directly. It is configured within an NPC asset file and managed by the NPC's `Role`. The `Role` is responsible for calling `computeSteering` on each server tick to drive the NPC's movement.

```java
// This logic is handled internally by the Role system.
// A developer does not typically write this code.

// Inside the Role's update loop:
Steering desiredSteering = new Steering();
activeBodyMotion.computeSteering(ref, this, sensorInfo, dt, desiredSteering, accessor);

// The Role then passes the populated 'desiredSteering' object
// to the MotionController to be translated into physical forces.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BodyMotionWanderBase()`. The component must be configured via its Builder during asset definition to ensure all parameters are correctly initialized.
-   **External State Modification:** Do not modify the internal state fields like `state` or `walkTime` from outside the class. This will break the state machine and lead to unpredictable behavior.
-   **Concurrent Access:** Do not call any methods on this component from an asynchronous task or a different thread. All interactions must be synchronized with the main server tick.

## Data Pipeline

The component transforms a high-level goal ("wander") into low-level steering instructions. It acts as a bridge between the AI's intent and the physics simulation.

> Flow:
> Server Tick -> NPC Role Update -> **BodyMotionWanderBase.computeSteering()** -> Queries Physics World via `MotionController.probeMove()` -> **BodyMotionWanderBase** (selects best direction) -> Populates `Steering` object -> `MotionController` applies forces -> NPC `TransformComponent` is updated.

