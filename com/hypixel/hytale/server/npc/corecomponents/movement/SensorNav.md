---
description: Architectural reference for SensorNav
---

# SensorNav

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorNav extends SensorBase {
```

## Architecture & Concepts
The SensorNav class is a specific implementation of the SensorBase contract, designed to evaluate the navigational state of a Non-Player Character (NPC). Within Hytale's server-side AI framework, a Sensor acts as a conditional predicate—a component that returns true or false based on a specific set of criteria.

SensorNav's primary role is to query an NPC's active MotionController and determine if its current state matches a predefined set of conditions. These conditions, configured via asset files, allow AI designers to build sophisticated behavior trees and state machines. For example, a designer can use SensorNav to ask questions like:

*   Is the NPC currently in a *walking* or *running* state?
*   Has the NPC been idle for more than 5 seconds?
*   Is the NPC within 2 meters of its navigation target?

This class is a fundamental, data-driven building block for creating reactive and stateful NPC movement logic. It decouples the high-level AI behaviors (e.g., "Patrol Area") from the low-level movement mechanics (e.g., pathfinding state).

## Lifecycle & Ownership
- **Creation:** SensorNav instances are not created directly via code. They are instantiated by the server's asset loading pipeline, specifically through a corresponding builder, BuilderSensorNav. This builder reads configuration data from an NPC's definition file (e.g., a JSON or HOCON asset) and constructs an immutable SensorNav object.
- **Scope:** The lifetime of a SensorNav instance is tied to its parent AI asset. It is created once when the NPC's behavior definition is loaded into memory and persists for the entire duration that the asset is active. It is effectively a shared, read-only template.
- **Destruction:** The object is marked for garbage collection when the server unloads the corresponding NPC asset definition, for instance, when a world zone is unloaded or during a server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including navStates, throttleDuration, and targetDeltaSquared, are declared as final and are initialized only once during construction. This object holds no mutable state and does not change after it is created. It is purely a configuration data holder.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable design, a single SensorNav instance can be safely used by multiple NPC agents simultaneously across different server threads without any need for locks or synchronization. The matches method is a pure function whose result depends solely on its inputs.

## API Surface
The public contract is minimal, exposing only the core evaluation logic inherited from its base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Ref, Role, double, Store) | boolean | O(1) | Evaluates the NPC's current motion state against the sensor's configured conditions. Returns true if all criteria are met. |
| getSensorInfo() | InfoProvider | O(1) | Returns metadata for debugging. Currently returns null and is reserved for future use. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its configuration is defined declaratively in NPC asset files. The AI engine then invokes the sensor as part of a behavior tree or state machine update tick.

```java
// PSEUDO-CODE: Engine-level usage, not for direct implementation.
// This logic would exist inside a generic AI behavior node.

// 1. The node holds a reference to the configured sensor.
SensorNav navCondition = this.getConditionFromAsset();

// 2. During an AI update tick, the node evaluates the sensor.
boolean canProceed = navCondition.matches(npc.getEntityRef(), npc.getActiveRole(), deltaTime, world.getStore());

// 3. The result determines the AI's next action.
if (canProceed) {
    npc.getBehaviorController().transitionToState("Attack");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance using `new SensorNav()`. This would bypass the critical asset-driven configuration pipeline and result in a non-functional component. All instances must be created by the engine via BuilderSensorNav.
- **Stateful Logic:** Do not extend this class with the intent of adding mutable state. The Sensor contract assumes immutable, reusable components. Storing per-NPC data in a sensor instance will lead to severe concurrency bugs.

## Data Pipeline
The primary flow involves translating declarative asset data into a runtime conditional check.

> Flow:
> NPC Asset File (JSON/HOCON) -> BuilderSensorNav (Asset Deserialization) -> **SensorNav Instance** (In Memory) -> AI Behavior Node -> `matches()` call -> MotionController State -> `boolean` Result -> AI State Transition

