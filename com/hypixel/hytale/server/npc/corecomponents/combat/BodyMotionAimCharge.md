---
description: Architectural reference for BodyMotionAimCharge
---

# BodyMotionAimCharge

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Strategy Object

## Definition
```java
// Signature
public class BodyMotionAimCharge extends BodyMotionBase {
```

## Architecture & Concepts

BodyMotionAimCharge is a concrete implementation of the BodyMotionBase abstract class. It encapsulates a specific, high-level movement behavior for server-side Non-Player Characters (NPCs): aiming at a target and preparing for a charge attack.

This component operates within the server's NPC AI framework, which is built upon an Entity-Component-System (ECS) architecture. Its primary responsibility is not to directly manipulate an entity's position or rotation, but to calculate the *desired* steering output. It acts as a "brain" for a specific action, translating sensory input (target location) into a movement intention (a populated Steering object).

This calculated intention is then consumed by lower-level systems, such as the NPC's active motion controller, which is responsible for the physics-aware execution of the movement. This creates a clean separation of concerns:
1.  **Sensors:** Provide world-state information (e.g., InfoProvider).
2.  **Behavior (BodyMotionAimCharge):** Decides what to do based on sensory input.
3.  **Actuators (Motion Controllers):** Execute the decision within the constraints of the game world.

This class is a stateful component within an NPC's behavior tree or finite state machine, representing the "Aiming & Charging" state.

### Lifecycle & Ownership
-   **Creation:** BodyMotionAimCharge is not instantiated directly. It is constructed by the server's NPC asset pipeline via its corresponding builder, BuilderBodyMotionAimCharge. It is defined as part of an NPC's static asset definition and instantiated when the NPC is spawned into the world.
-   **Scope:** The object's lifetime is tightly coupled to its parent NPC entity. It persists as a component of the NPC's Role for as long as the NPC exists.
-   **Destruction:** The object is marked for garbage collection when the parent NPC entity is despawned or destroyed, and all references within the NPC's Role and behavior components are cleared.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains several fields like aimingData, direction, and probeMoveData which are modified during each computation cycle. These fields are reused across ticks to minimize memory allocation and garbage collection overhead. The relativeTurnSpeed field, however, is immutable after construction.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed concurrently.** It is designed to be operated exclusively by the server's main game loop thread on a per-NPC basis. All state is managed with the assumption of a single-threaded, sequential update cycle. Unsynchronized access from other threads will result in severe race conditions and undefined AI behavior.

## API Surface
The public contract is defined by its parent, BodyMotionBase, and is invoked by the NPC's core AI loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| preComputeSteering(...) | void | O(1) | The first phase of the AI tick. It shares internal state, specifically the AimingData, with the sensor system. This allows other systems to react to the NPC's aiming status. |
| computeSteering(...) | boolean | O(N) | The primary logic execution method. Calculates the required yaw and pitch to face a target, validates the charge path, and populates the output Steering object. Complexity is dominated by the probeMove call. |

## Integration Patterns

### Standard Usage
BodyMotionAimCharge is managed by a higher-level AI component, typically the NPC's Role. The Role selects this behavior based on the NPC's current state and invokes its methods once per server tick. The developer does not interact with this class directly but rather configures it through NPC asset files.

```java
// Hypothetical invocation from within an NPC's Role update method

// The currentMotion is an instance of BodyMotionAimCharge
// This is a conceptual example of the engine's internal loop

// Phase 1: Pre-computation and data sharing
currentMotion.preComputeSteering(entityRef, this, sensorInfo, store);

// Phase 2: Core logic execution
boolean canSteer = currentMotion.computeSteering(
    entityRef,
    this,
    sensorInfo,
    deltaTime,
    outputSteering, // The object to be populated
    componentAccessor
);

// The outputSteering object is now used by a motion controller
// to physically turn and move the NPC.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BodyMotionAimCharge()`. The object is stateful and requires specific configuration provided by the asset pipeline and its associated builder. Direct instantiation will result in a misconfigured and non-functional component.
-   **External State Mutation:** Do not modify the internal AimingData object from outside this class. The internal state is tightly managed and external changes will break the aiming and charging logic.
-   **Asynchronous Execution:** Do not call computeSteering from a separate thread or outside the main server tick. The class relies on a consistent world state provided by the synchronous game loop.

## Data Pipeline
The flow of data through this component during a single server tick is linear and predictable. It transforms high-level sensory data into a low-level steering command.

> Flow:
> InfoProvider (Sensor) -> Target Position Vector -> **BodyMotionAimCharge** -> Yaw/Pitch Calculation -> Steering Object -> NPC Motion Controller -> Final TransformComponent Update

