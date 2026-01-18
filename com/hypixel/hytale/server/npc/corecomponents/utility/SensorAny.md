---
description: Architectural reference for SensorAny
---

# SensorAny

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Component / Transient

## Definition
```java
// Signature
public class SensorAny extends SensorBase {
```

## Architecture & Concepts
The **SensorAny** class is a fundamental component within the server-side NPC AI framework. It serves as a concrete implementation of the **SensorBase** abstraction, designed to represent a universally true condition.

Unlike specialized sensors that detect players, world states, or entity attributes, **SensorAny** provides no specific environmental information. Its primary architectural role is to act as a default or unconditional trigger within a larger decision-making structure, such as a Behavior Tree or a Finite State Machine. By always evaluating as "active" or "true", it enables AI designers to create logic paths that are not contingent on any specific world state, serving as a guaranteed entry point for a sequence of actions or a fallback behavior.

This component is a leaf node in the AI's sensory system, intentionally designed to be simple and computationally inexpensive.

### Lifecycle & Ownership
- **Creation:** An instance of **SensorAny** is not created directly. It is instantiated by the NPC AI system, typically through a data-driven configuration (e.g., a JSON definition of an NPC's behavior). The system uses a **BuilderSensorBase** to construct and initialize the sensor when its parent NPC entity is spawned.
- **Scope:** The lifecycle of a **SensorAny** instance is tightly coupled to its owning NPC entity. It persists as long as the NPC is active in the game world.
- **Destruction:** The object is marked for garbage collection when the parent NPC is despawned or destroyed. There is no explicit destruction method; its cleanup is managed by the JVM.

## Internal State & Concurrency
- **State:** This class is inherently stateless. It contains no member fields and its behavior is constant. However, it inherits from **SensorBase**, which may contain mutable state related to cooldowns or activation status. Therefore, the composite object should be treated as **Mutable**.
- **Thread Safety:** **Not Thread-Safe.** Like most game engine components, all interactions with an NPC's AI components, including **SensorAny**, must be confined to the main server tick thread for the world or region in which the NPC exists. Unsynchronized access from other threads will lead to race conditions and unpredictable AI behavior.

## API Surface
The public contract is minimal, reflecting its simple, specialized purpose. The constructor is public for framework access but should not be invoked by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSensorInfo() | InfoProvider | O(1) | **Always returns null.** This is the defining behavior of the class, indicating it provides no specific world data. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. Its usage is declared within an NPC's AI configuration data. The AI system will manage its lifecycle and evaluation. The following conceptual example shows how the system might use a configured sensor.

```java
// PSEUDOCODE: Internal AI system evaluating its sensors
// A developer would not write this code.

// The system retrieves the pre-configured sensor from the NPC
SensorBase currentSensor = npc.getBehaviorTree().getActiveSensor();

// The system checks the sensor's output
if (currentSensor instanceof SensorAny) {
    // This block will always execute because SensorAny is always "true"
    npc.getBehaviorTree().triggerUnconditionalAction();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SensorAny(builder)`. This bypasses the managed lifecycle and configuration handled by the NPC's AI factory. All sensors must be instantiated by the framework.
- **Checking for Information:** Do not call `getSensorInfo()` and expect a non-null result. The core design principle of this class is that it provides no data. Code that relies on its `InfoProvider` will fail with a NullPointerException.

## Data Pipeline
**SensorAny** does not process data. Instead, it acts as a control flow primitive within the NPC decision-making pipeline.

> Flow:
> NPC AI Tick -> Behavior Tree Evaluation -> **SensorAny Node** -> Unconditional "True" Result -> Associated Action or State Transition is Triggered

