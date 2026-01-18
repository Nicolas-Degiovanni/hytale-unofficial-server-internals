---
description: Architectural reference for SensorTarget
---

# SensorTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorTarget extends SensorWithEntityFilters {
```

## Architecture & Concepts
The SensorTarget is a perception component within the Hytale NPC AI framework. It is not designed to find new targets, but rather to **validate an existing target** that an NPC has already acquired and stored.

In the context of a Behavior Tree, a Sensor acts as a conditional node. The SensorTarget specifically answers the question: "Is the entity I am currently targeting still a valid target?". It performs this validation based on a combination of range checks and a configurable set of entity filters.

Its primary role is to act as a guard in AI logic. For example, a "Chase" behavior would be preceded by a SensorTarget check to ensure the NPC does not continue chasing an entity that has moved out of range, become invisible, or no longer meets the required criteria (e.g., is no longer hostile).

The component integrates tightly with the **Role** and **MarkedEntitySupport** systems. An NPC's Role holds its current state, and MarkedEntitySupport is the memory system where references to important entities, like a combat target or a follow-leader, are stored in numbered slots. SensorTarget reads from one of these slots, identified by its configured **targetSlot**.

## Lifecycle & Ownership
-   **Creation:** SensorTarget is instantiated by the server's NPC asset loading pipeline when an NPC's behavior is being constructed. It is configured via a corresponding **BuilderSensorTarget**, which reads parameters from the NPC's definition files (e.g., JSON or HOCON assets). It is not created dynamically during gameplay.
-   **Scope:** An instance of SensorTarget is owned by an NPC's **Role**. Its lifetime is bound to the lifetime of that specific NPC entity in the world.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or destroyed, and its associated Role is de-referenced.

## Internal State & Concurrency
-   **State:** The component is primarily configured with immutable state. Fields like **targetSlot**, **range**, and **autoUnlockTarget** are final and set during construction. However, it contains a mutable **EntityPositionProvider** field, which caches the target's position upon a successful validation. This provider's state is cleared or updated each time the **matches** method is invoked.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the server's main entity update thread. The NPC AI system operates in a single-threaded model per world or region. Concurrent calls to **matches** would lead to race conditions when accessing the shared entity **Store** and updating the internal **positionProvider**.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Core evaluation method. Returns true if the entity in the configured target slot is still valid. N is the number of configured entity filters. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal position provider, allowing subsequent AI nodes to access the validated target's location without re-fetching it. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly in procedural code. It is defined declaratively within an NPC's asset files and executed by the AI's Behavior Tree processor. The system internally invokes the **matches** method each tick that the AI logic evaluates this sensor.

A conceptual representation of its use by the engine:
```java
// Engine's AI Tick (Conceptual)
// Assume 'sensor' is the SensorTarget instance for the current NPC

boolean isTargetStillValid = sensor.matches(npcRef, npcRole, deltaTime, worldStore);

if (isTargetStillValid) {
    // Proceed with behavior, e.g., AttackNode
    // The AttackNode can now use sensor.getSensorInfo() to get the target's position
} else {
    // Fail this branch of the behavior tree, forcing the AI to re-evaluate
    // and potentially find a new target.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new SensorTarget()**. The object is complex to configure and must be constructed by the asset loading system via its corresponding Builder. Manual creation will result in a misconfigured and non-functional sensor.
-   **State Tampering:** Do not manually access or modify the internal **positionProvider**. Its state is managed exclusively by the **matches** method as part of the evaluation lifecycle.
-   **Misconfiguration of autoUnlockTarget:** Setting **autoUnlockTarget** to false can cause AI lock-ups. If a target becomes invalid (e.g., moves out of range), the sensor will correctly return false, but the NPC will never clear its target slot. It may fail to acquire new targets because it believes it is still "locked on" to the now-invalid one.

## Data Pipeline
SensorTarget acts as a conditional gate in the AI's decision-making process. Its data flow is focused on validation rather than transformation.

> Flow:
> AI Behavior Tree Tick -> **SensorTarget.matches()**
> 1.  Read **targetSlot** from own configuration.
> 2.  Read target **Ref<EntityStore>** from **Role.getMarkedEntitySupport()**.
> 3.  Read own and target's **TransformComponent** from the global **Store**.
> 4.  Perform distance check against configured **range**.
> 5.  Execute entity filters against the target.
> 6.  **[IF INVALID]** If **autoUnlockTarget** is true, write a "clear" command to **Role.getMarkedEntitySupport()**.
> 7.  **[IF VALID]** Write the target's position into the internal **EntityPositionProvider**.
> 8.  Return boolean result to Behavior Tree.

