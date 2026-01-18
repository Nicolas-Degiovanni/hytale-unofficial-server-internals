---
description: Architectural reference for BodyMotionNothing
---

# BodyMotionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionNothing extends BodyMotionBase {
```

## Architecture & Concepts
BodyMotionNothing is a concrete implementation of the **Strategy Pattern** for NPC movement. It represents the "Null Object" pattern within the motion control system. Its sole purpose is to provide a valid, non-null motion component that actively brings an NPC to a complete stop.

This component is fundamental to the server's AI state machine or behavior tree. When an NPC's logic dictates it should be idle, waiting, stunned, or otherwise stationary, its active motion strategy is switched to an instance of BodyMotionNothing. This design elegantly avoids null checks and special case logic in the higher-level NPC update loop. Instead of checking if an NPC *has* a motion component, the system can polymorphically call computeSteering on the active component, and BodyMotionNothing ensures the resulting steering command is zero.

It is the simplest possible implementation of the BodyMotionBase contract, serving as a baseline and a critical tool for managing NPC states.

### Lifecycle & Ownership
- **Creation:** Instantiated by higher-level AI constructs, such as a Role or a BehaviorTree node, when an NPC transitions into a state that requires no movement. It is configured via a BuilderBodyMotionBase, which sets base properties like priority, even though this specific implementation does not use them for complex calculations.
- **Scope:** The lifetime of a BodyMotionNothing instance is ephemeral and tied directly to the NPC's current behavioral state. It persists only as long as the NPC is intended to be stationary.
- **Destruction:** The object is eligible for garbage collection as soon as the NPC's state machine transitions to a new state and replaces it with a different BodyMotionBase implementation (e.g., BodyMotionSeek, BodyMotionFlee).

## Internal State & Concurrency
- **State:** BodyMotionNothing is effectively stateless in its own right. It inherits stateful properties from BodyMotionBase but its core logic in computeSteering does not depend on or modify any internal fields. Its behavior is constant and predictable.
- **Thread Safety:** This class is not thread-safe by design. The computeSteering method is expected to be invoked exclusively from the server's main NPC update thread during a single game tick. It directly mutates the passed-in Steering object, a pattern that is inherently unsafe if called from multiple threads concurrently for the same NPC. All interactions should be synchronized with the server's tick lifecycle.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Resets the provided Steering object to a zero state (no linear or angular velocity). Always returns true to indicate success. |

## Integration Patterns

### Standard Usage
A developer typically does not interact with this class directly. Instead, it is assigned as the active motion strategy by a state management system.

```java
// Example from within an NPC's Role or Behavior logic

// When an NPC's behavior dictates it should stop moving, its motion
// component is replaced with a BodyMotionNothing instance.
BuilderBodyMotionBase motionBuilder = new BuilderBodyMotionBase(context);
BodyMotionBase idleMotion = new BodyMotionNothing(motionBuilder);

// The NPC's state manager holds this reference.
npc.getActiveRole().setBodyMotion(idleMotion);

// The server's NPC update system will then automatically call computeSteering
// on this component during the next tick, causing the NPC to halt.
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Pausing:** Do not use BodyMotionNothing to temporarily pause a complex, ongoing movement like pathfinding. This component completely clears all steering intentions. Using it will discard the previous steering goal, and resuming the old behavior will require a full recalculation. Use a dedicated pause mechanism if one exists.
- **Direct Invocation:** Never call the computeSteering method directly. It is a core part of the NPC update lifecycle and is managed by the server's AI processing systems. Manual invocation will bypass priority blending and other essential parts of the motion arbitration pipeline, leading to unpredictable behavior.

## Data Pipeline
BodyMotionNothing acts as a data "sink" or "zero-generator" in the motion pipeline. It does not process incoming data but rather produces a definitive output that halts further motion calculations.

> Flow:
> AI Behavior System -> State Transition to "Idle" -> **BodyMotionNothing** becomes active motion component -> NPC Update System invokes `computeSteering` -> `Steering` object is cleared -> Physics System receives zero-vector steering command -> NPC's velocity is dampened to zero.

