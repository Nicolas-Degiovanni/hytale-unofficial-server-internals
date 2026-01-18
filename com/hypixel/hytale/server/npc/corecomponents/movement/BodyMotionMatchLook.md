---
description: Architectural reference for BodyMotionMatchLook
---

# BodyMotionMatchLook

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionMatchLook extends BodyMotionBase {
```

## Architecture & Concepts
BodyMotionMatchLook is a concrete implementation of the **Strategy** pattern for NPC movement. It represents a specific, atomic steering behavior: aligning the NPC's body orientation with its current head orientation.

Within the Hytale NPC AI framework, this class acts as a translator, converting sensory information (where the NPC is looking) into a low-level motor command (how the NPC's body should turn). It is a fundamental building block used in higher-level AI constructs like Behavior Trees or Finite State Machines.

Its primary role is to ensure that an NPC's body "catches up" to its head yaw. This is critical for creating believable character motion, for example, when an NPC turns its head to look at a point of interest and then turns its entire body to face it. It decouples the act of looking from the act of pathfinding, allowing for more complex and layered behaviors.

## Lifecycle & Ownership
- **Creation:** Instantiated via its corresponding builder, BuilderBodyMotionBase, during the assembly of an NPC's AI profile. This process is typically data-driven, configured through server assets rather than direct Java code. It is never created standalone.
- **Scope:** The lifetime of a BodyMotionMatchLook instance is tied to the AI state that requires it. It may be short-lived, existing only for the duration of a single "face target" action, or it may persist as part of a default, idle behavior within an NPC's Role.
- **Destruction:** The object is eligible for garbage collection as soon as the parent AI component (e.g., a Role, a behavior node) that holds a reference to it is deactivated or replaced. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no internal fields and does not cache data across invocations. Its behavior is purely a function of the arguments passed to its computeSteering method.

- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, it is designed to operate exclusively within the server's main game loop thread. The components it interacts with, such as EntityStore and ComponentAccessor, are **not thread-safe**.

    **WARNING:** Invoking computeSteering from any thread other than the one managing the NPC's tick update will lead to race conditions, data corruption, and server instability. All interactions must be synchronized with the main server tick.

## API Surface
The public contract is minimal, consisting of the single behavior method inherited from its base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Modifies the output parameter *desiredSteering* to set the target yaw based on the entity's HeadRotation component. Always returns true. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by high-level game logic. It is a pluggable component consumed by the NPC's core AI update loop, typically managed by a Role. The system selects this BodyMotion strategy when the NPC's current state dictates that it should align its body with its gaze.

```java
// Conceptual example of how the engine uses this component.
// This code would reside within a higher-level AI controller, like a Role.

// On a given server tick for an NPC:
Steering outputSteering = new Steering();
BodyMotionBase currentMotion = npc.getActiveBodyMotion(); // This could be an instance of BodyMotionMatchLook

// The engine invokes the component to calculate the desired movement
boolean success = currentMotion.computeSteering(
    npc.getEntityRef(),
    npc.getActiveRole(),
    npc.getSensorInfo(),
    deltaTime,
    outputSteering,
    world.getComponentAccessor()
);

// The resulting outputSteering is then passed to the physics and animation systems.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BodyMotionMatchLook()`. The object must be constructed via its designated builder to ensure the base class is correctly initialized. Failure to do so will bypass critical setup logic.
- **Missing Dependencies:** Attempting to use this component on an entity that does not possess a HeadRotation component will trigger a runtime assertion failure. This is by design to enforce system integrity. Ensure all required components are present in the entity's archetype.
- **Stateful Subclassing:** Do not extend this class to add state. BodyMotion components are designed to be stateless, reusable strategies. Any required state should be stored in dedicated entity components, which this class can then read during its execution.

## Data Pipeline
BodyMotionMatchLook functions as a simple processing node in the NPC's per-tick data flow. It reads from the Entity-Component System and writes to a transient steering object.

> Flow:
> AI Behavior System (updates HeadRotation) -> EntityStore (persists HeadRotation state) -> **BodyMotionMatchLook** (reads head yaw) -> Steering object (receives desired body yaw) -> NPC Movement System (applies steering to entity's velocity and orientation)

