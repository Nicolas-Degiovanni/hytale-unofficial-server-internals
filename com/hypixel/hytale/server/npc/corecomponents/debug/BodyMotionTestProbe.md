---
description: Architectural reference for BodyMotionTestProbe
---

# BodyMotionTestProbe

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug
**Type:** Transient Component

## Definition
```java
// Signature
public class BodyMotionTestProbe extends BodyMotionBase {
```

## Architecture & Concepts
The BodyMotionTestProbe is a specialized diagnostic component within the Non-Player Character (NPC) AI framework. As an extension of BodyMotionBase, it represents a concrete strategy for controlling an NPC's physical orientation and intended movement. Its primary function is not to facilitate complex, dynamic navigation but to serve as a debugging tool for developers to analyze the server's core movement and collision systems.

This component operates by taking a target direction, typically from a sensor, and then querying the underlying physics system via a "probe". This probe determines how far an NPC *could* travel along that vector before an obstruction is met. The results, including the viable travel distance and direction, are then formatted and displayed as on-screen debug text.

Key architectural features include:
- **Diagnostic Focus:** It exists entirely within the `debug` package and leverages `RoleDebugFlags` for its operation. It is not intended for use in production NPC behaviors.
- **Physics Abstraction:** It does not implement collision logic itself. Instead, it acts as a client to the active `MotionController`, delegating the complex task of world intersection testing through the `probeMove` method.
- **Configurability:** Instantiated via a builder, it allows developers to configure initial position adjustments, movement constraints, and angle snapping to create highly specific and repeatable test scenarios.

## Lifecycle & Ownership
- **Creation:** An instance of BodyMotionTestProbe is created by the NPC's `Role` system when a behavior requiring it becomes active. It is configured and instantiated using a corresponding `BuilderBodyMotionTestProbe`, which is typically defined in NPC asset files.
- **Scope:** The object's lifetime is tightly coupled to the NPC's active `Role`. It persists only as long as the NPC is executing the specific debug behavior that utilizes this component.
- **Destruction:** The object is eligible for garbage collection once the parent `Role` is deactivated or the associated `NPCEntity` is removed from the world. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** BodyMotionTestProbe is a stateful component. It maintains configuration data from its builder (e.g., `adjustDistance`, `snapAngle`) and caches runtime objects like the `direction` vector and the `probeMoveData` container. The `probeMoveData` object is mutable and accumulates data about movement segments.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be owned and operated exclusively by the server's main tick thread for the world in which the NPC resides. All method calls must be synchronized with the server's update loop. Concurrent access to its methods, particularly `computeSteering`, will lead to race conditions and corrupt internal state.

## API Surface
The public contract is defined by its overrides of the BodyMotionBase class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(...) | void | O(1) | Initializes the component for use. Teleports the NPC to a starting test position if configured and caches the debug display flag. Must be called before `computeSteering`. |
| computeSteering(...) | boolean | O(N) | The primary logic method. Calculates the steering direction, invokes the motion controller's `probeMove` function, and updates the debug display. The complexity is dominated by the `probeMove` call, which performs physics queries against the world. |

## Integration Patterns

### Standard Usage
This component is not intended to be instantiated or called directly in code. It is designed to be configured within an NPC's behavior assets and managed automatically by the `Role` system. The engine's NPC update loop invokes `computeSteering` on each tick for the active `BodyMotion` component.

```java
// This logic is representative of the engine's internal NPC update loop.
// A developer would not write this code.

// Inside an NPC's Role update:
Steering desiredSteering = new Steering();
BodyMotionTestProbe motion = npc.getActiveBodyMotion(); // Assumes this is the active component
boolean success = motion.computeSteering(ref, role, sensorInfo, dt, desiredSteering, accessor);

// The engine would then use desiredSteering to influence the NPC's transform.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BodyMotionTestProbe()`. The component must be created via its builder and managed by the NPC's `Role` to ensure its lifecycle and dependencies are handled correctly.
- **Stateful Misuse:** The internal `direction` vector and `probeMoveData` are reused across calls to `computeSteering`. Do not attempt to access or modify this component from any thread other than the main server tick thread for that NPC.
- **Skipping Activation:** Calling `computeSteering` before `activate` has been invoked by the `Role` system can result in incorrect initial positioning and behavior.

## Data Pipeline
The component transforms a target position into a debuggable movement probe result.

> Flow:
> Target Position (from `InfoProvider`) -> `computeSteering` calculates direction vector -> Vector is optionally snapped to grid (`snapAngle`) -> `Steering` object is populated -> `MotionController.probeMove` is called with the direction vector -> Physics system performs collision checks -> Resulting travel distance is returned -> **BodyMotionTestProbe** formats distance into a debug string -> `RoleDebugSupport.setDisplayCustomString` is called -> Debug text is rendered for the client.

