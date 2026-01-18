---
description: Architectural reference for SensorCombatActionEvaluator
---

# SensorCombatActionEvaluator

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorCombatActionEvaluator extends SensorBase {
```

## Architecture & Concepts
The SensorCombatActionEvaluator is a specialized component within the server-side NPC AI framework. It functions as a conditional gate, or "Sensor," used by an NPC's state machine to make decisions during combat. Its primary role is to determine if the NPC's current combat target is positioned within a dynamically configured range.

This component is not a simple static distance check. It is designed for data-driven AI behaviors. Instead of hard-coded ranges, it reads the minimum range, maximum range, and a desired positioning angle from the NPC's **ValueStore** component at runtime. This allows AI designers to modify an NPC's combat engagement envelope through behaviors, scripts, or other game systems without altering the core sensor logic.

During its initialization, this sensor populates a **CombatConstructionData** component on the NPC entity. This acts as a registration step, signaling to other related combat systems which ValueStore slots contain the authoritative range and angle data for the current AI state. This decouples the sensor from the systems that *use* its data, forming a cohesive but loosely coupled combat evaluation pipeline.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the NPC asset loading pipeline via its corresponding builder, **BuilderSensorCombatActionEvaluator**. It is constructed when an NPC's AI behavior tree, which includes this sensor, is loaded and initialized. The **BuilderSupport** context provides critical links to entity data and parameter slot mappings from the asset definition.
- **Scope:** The lifetime of a SensorCombatActionEvaluator instance is tied directly to the AI state that defines it. It persists only as long as its parent AI state is active or potentially active in the behavior tree.
- **Destruction:** The object is eligible for garbage collection once the NPC transitions to a different AI state that does not use this sensor. There is no explicit destruction method. Internal state, such as target information, is cleared at the beginning of each evaluation cycle within the **matches** method.

## Internal State & Concurrency
- **State:** This class is highly stateful but manages its state on a per-tick basis. It holds immutable configuration like **targetInRange** and **allowableDeviation**, and mutable slot indices for reading from the ValueStore. Its internal helper objects, **EntityPositionProvider** and **MultipleParameterProvider**, are stateful and are explicitly updated or cleared during each call to **matches**.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread responsible for its parent NPC's AI updates. The **matches** method performs sequential, state-mutating operations. Concurrent access would lead to severe race conditions and unpredictable AI behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | The core evaluation method. Returns true if the combat target meets the range criteria defined in the ValueStore. Throws exceptions if required components are missing. |
| getSensorInfo() | InfoProvider | O(1) | Provides access to contextual information, primarily the target's position, for other AI systems to consume. |

## Integration Patterns

### Standard Usage
This component is not intended for direct manual use in code. It is configured within NPC asset files and managed entirely by the AI runtime. An AI designer defines the sensor's parameters and which ValueStore slots to read from. The system then invokes it automatically.

The conceptual flow within the AI engine is as follows:
```java
// Engine-level code, not for typical developer use.
// The engine iterates through the active state's sensors.
boolean conditionsMet = sensor.matches(npcRef, npcRole, deltaTime, entityStore);

if (conditionsMet) {
    // Allow the NPC to proceed with the associated combat action.
    executeCombatBehavior();
} else {
    // Block the action or transition to a different AI state.
    findNewBehavior();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorCombatActionEvaluator()`. The constructor requires internal builder context that is only available during asset loading. Attempting to create it manually will fail and violates the design of the AI framework.
- **External State Mutation:** Do not retrieve and modify the internal **positionProvider** or **parameterProvider** objects. Their state is tightly managed by the **matches** method and external changes will corrupt the sensor's evaluation logic.
- **Ignoring ValueStore:** This sensor is useless if the target NPC does not have a ValueStore component, or if the configured slots are not populated with valid range data. The sensor will fail to read valid parameters, likely resulting in default or zero values and incorrect behavior.

## Data Pipeline
The flow of data for a single evaluation is a read-only process from the perspective of the wider Entity-Component-System.

> Flow:
> AI State Machine Tick -> **SensorCombatActionEvaluator.matches()** -> Read Target from **Role** -> Read Range/Angle from **ValueStore** -> Read Positions from **TransformComponent** -> Boolean Result -> AI State Machine Decision

