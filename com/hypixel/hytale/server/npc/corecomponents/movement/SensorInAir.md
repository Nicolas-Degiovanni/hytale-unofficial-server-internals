---
description: Architectural reference for SensorInAir
---

# SensorInAir

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Component / Transient

## Definition
```java
// Signature
public class SensorInAir extends SensorBase {
```

## Architecture & Concepts
SensorInAir is a concrete implementation of the SensorBase abstraction, designed to function as a conditional predicate within the server-side NPC Artificial Intelligence framework. Its sole responsibility is to determine if an NPC entity is currently airborne.

This component is a fundamental building block in creating reactive and environmentally-aware NPC behaviors. It acts as a gateway in a Behavior Tree or a condition in a Finite State Machine, allowing the AI to select different actions or states based on the NPC's physical interaction with the world. For example, an NPC might transition to a "flailing" or "falling" animation state when this sensor returns true.

Architecturally, SensorInAir bridges the AI system with the physics and movement system. It queries the NPC's active MotionController, which is the component responsible for managing the entity's physical state, including velocity, gravity, and collision. The stateless nature of this sensor makes it highly efficient and reusable across countless NPC definitions.

## Lifecycle & Ownership
- **Creation:** SensorInAir is not intended for direct instantiation. It is constructed by the NPC AI system, typically during the loading and parsing of an NPC's behavior definition from a configuration file. The process uses the BuilderSensorBase, ensuring that all dependencies and base properties are correctly initialized.
- **Scope:** The lifetime of a SensorInAir instance is tightly coupled to the NPC's AI definition. It persists as a small, stateless object within the AI's structural graph (e.g., a Behavior Tree node) for as long as the parent NPC entity exists.
- **Destruction:** The object is eligible for garbage collection when the NPC entity is unloaded or its AI definition is replaced. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable** post-construction. It contains no internal fields to store data between calls. All necessary context, such as the NPC's Role and the game world Store, is provided as arguments to the *matches* method.
- **Thread Safety:** The component is inherently thread-safe due to its stateless design. However, it is expected to be invoked exclusively from the main server thread responsible for the NPC's AI update tick. The underlying components it accesses, such as Role and MotionController, are not guaranteed to be safe for concurrent modification from other threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC is airborne. Returns true only if the base sensor conditions pass AND the NPC's MotionController reports an *inAir* state. |
| getSensorInfo() | InfoProvider | O(1) | Returns null. This sensor does not provide extended debugging information. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in code. Instead, it is declared within an NPC's data-driven behavior configuration. The AI system then instantiates and integrates it into the decision-making loop.

The system's internal usage pattern is as follows:
```java
// PSEUDOCODE: How the AI engine uses the sensor
// This code would exist within a Behavior Tree node or similar AI construct.

boolean isConditionMet = sensorInAirInstance.matches(npcRef, npcRole, deltaTime, worldStore);

if (isConditionMet) {
    // Execute the "In Air" branch of the behavior tree.
    // For example, switch to a falling animation or a glide ability.
} else {
    // Execute the "On Ground" branch.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Under no circumstances should you use `new SensorInAir()`. The component relies on the builder pattern for proper initialization within the AI framework. Bypassing the framework will lead to an uninitialized and non-functional sensor.
- **Incorrect Entity Type:** Attaching this sensor to an entity that does not possess a MotionController will result in runtime errors, likely a NullPointerException, when the *matches* method is invoked. This sensor is strictly for entities governed by the server's physics and movement systems.

## Data Pipeline
SensorInAir acts as a conditional gate in the AI's decision-making flow, not as a data processor. Its position in the pipeline is to interpret physical state and convert it into a boolean signal for the behavior system.

> Flow:
> AI Update Tick -> Behavior Tree Node Evaluation -> **SensorInAir.matches()** queries MotionController -> Boolean Result -> AI selects appropriate action (e.g., Play Fall Animation)

