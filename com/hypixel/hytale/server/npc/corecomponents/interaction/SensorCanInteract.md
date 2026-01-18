---
description: Architectural reference for SensorCanInteract
---

# SensorCanInteract

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction
**Type:** Component / Strategy Object

## Definition
```java
// Signature
public class SensorCanInteract extends SensorBase {
```

## Architecture & Concepts
The SensorCanInteract class is a specific implementation of the Sensor pattern within the Hytale NPC AI framework. Its primary function is to act as a conditional predicate, answering the question: "Can the NPC currently perceive and socially engage with its designated interaction target?"

This sensor is a critical component in behavior trees or finite state machines that govern NPC social interactions, such as initiating dialogue, trading, or quest-related actions. It logically gates these higher-level behaviors, ensuring they only activate under valid conditions.

Architecturally, it serves as a bridge between three core domains:
1.  **World State:** It queries the entity-component store for physical data like position and orientation.
2.  **AI State:** It retrieves the NPC's current focus, the *interactionIterationTarget*, from the NPC's Role.
3.  **Social/Faction System:** It uses the Attitude system to determine the relationship between the NPC and its target.

By encapsulating these checks, SensorCanInteract decouples complex AI behaviors from the low-level details of world simulation and social state management.

### Lifecycle & Ownership
-   **Creation:** This object is not instantiated directly at runtime. It is constructed by the server's asset loading pipeline, specifically via a BuilderSensorCanInteract, when an NPC archetype is loaded from its definition files. Its configuration, such as the view cone and valid attitudes, is deserialized from these files.
-   **Scope:** The object is immutable and shared. A single SensorCanInteract instance exists for each unique NPC definition and is used by all NPC entities of that type. Its lifetime is tied to the lifetime of the NPC archetype in server memory.
-   **Destruction:** The object is garbage collected when its corresponding NPC archetype is unloaded, typically during a server shutdown or a hot-reload of game assets. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. The core state, including the viewCone and attitudes set, is declared as final and injected during construction. The class holds no mutable state related to a specific NPC instance or game tick. All dynamic data required for its evaluation is passed as arguments to the *matches* method.
-   **Thread Safety:** **Thread-safe**. Due to its immutable nature, a single instance can be safely accessed by multiple threads concurrently without locks or synchronization. The responsibility for ensuring thread-safe access to the *Store* and *Ref* objects passed into its methods lies with the calling AI scheduler or update loop.

## API Surface
The public contract is minimal, focusing entirely on the Sensor interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC can interact with its target. Returns true if the target is alive, within the view cone, and has a valid attitude. |
| registerWithSupport(role) | void | O(1) | A lifecycle callback used to declare system dependencies. It signals to the WorldSupport that the Attitude cache is required for this sensor to function. |

## Integration Patterns

### Standard Usage
A game developer or designer does not interact with this class directly in code. Instead, it is declared and configured within an NPC's asset definition file. The AI engine then invokes it automatically as part of a behavior tree's conditional evaluation.

A conceptual NPC definition might look like this:

```yaml
# Example NPC Asset Definition (Conceptual)
archetype: "villager"
ai:
  behaviors:
    - type: "InteractWithPlayer"
      conditions:
        - sensor: "CanInteract"
          params:
            viewSector: 120 # degrees
            attitudes: [FRIENDLY, NEUTRAL]
```

The underlying system then uses the configured sensor instance:

```java
// Simplified engine code showing how the sensor is used internally
// This code is part of the AI scheduler, not user-written game logic.

// For a given NPC's Role...
SensorCanInteract sensor = role.getActiveBehavior().getConditionSensor();
boolean canProceed = sensor.matches(npcRef, role, deltaTime, worldStore);

if (canProceed) {
    // Execute the interaction behavior (e.g., open dialogue UI)
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorCanInteract(...)`. The object is tightly coupled to the asset loading pipeline for its configuration. Manual creation will result in an unconfigured and non-functional sensor.
-   **Stateful Modification:** Do not attempt to modify this class to hold per-NPC state. This would violate its shared, immutable design and introduce severe concurrency issues.
-   **Ignoring Dependencies:** The call to `registerWithSupport` is a critical part of the NPC's initialization. If this sensor is used in a custom AI system that bypasses the standard Role setup, the required Attitude cache may not be populated, leading to incorrect behavior or runtime errors.

## Data Pipeline
The `matches` method orchestrates a flow of data from multiple systems to produce a single boolean result.

> Flow:
> AI Behavior Tree Tick -> `Role.getStateSupport().getInteractionIterationTarget()` -> **SensorCanInteract.matches()** -> Query Entity Store for `Transform`, `HeadRotation`, `DeathComponent` -> Query `WorldSupport` for `Attitude` -> Geometric & Set Logic -> **Boolean Result** -> Behavior Tree Execution Path

