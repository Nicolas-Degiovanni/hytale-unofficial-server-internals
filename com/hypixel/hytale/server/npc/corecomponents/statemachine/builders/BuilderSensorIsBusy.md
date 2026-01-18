---
description: Architectural reference for BuilderSensorIsBusy
---

# BuilderSensorIsBusy

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorIsBusy extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorIsBusy class is a concrete implementation of the **Factory Pattern**, operating exclusively within the server-side NPC AI asset pipeline. Its sole responsibility is to instantiate a runtime `SensorIsBusy` object, which is a component used by an NPC's state machine to evaluate a specific condition—namely, whether the NPC is currently in a "busy" state.

This class serves as a critical bridge between static data and live game objects. NPC behaviors are defined declaratively in asset files (e.g., JSON). The asset loading system discovers a definition for a "SensorIsBusy" and uses this builder class to translate that data into a functional, in-memory `Sensor` object. This decouples the asset format from the runtime implementation, allowing game designers to compose complex AI behaviors without writing code, and allowing engineers to evolve the sensor's implementation without breaking asset definitions.

The `BuilderDescriptorState.Stable` return value signals to the asset pipeline and associated tooling that this component is considered production-ready and is not experimental.

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the NPC asset deserialization service when it parses an NPC behavior tree or state machine definition from a data file. It is mapped from a string identifier within the asset.
-   **Scope:** Extremely short-lived and ephemeral. An instance of BuilderSensorIsBusy exists only for the fraction of a second required to call its `build` method.
-   **Destruction:** The object is immediately eligible for garbage collection after the `build` method returns the `SensorIsBusy` product. It holds no persistent state and is not registered in any long-term context.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no member fields and its logic is entirely self-contained. Any configuration required for the resulting `Sensor` is passed down from its parent, `BuilderSensorBase`, which is populated by the deserializer.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless factory, it can be safely invoked by multiple threads during a parallelized asset loading process without any risk of race conditions or need for synchronization primitives.

## API Surface
The public contract is minimal, focusing entirely on metadata and object creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | **Primary Factory Method.** Constructs and returns a new SensorIsBusy instance. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for use in development tools and editors. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component, signaling its readiness for use. |

## Integration Patterns
This class is not intended for direct use in gameplay code. Developers interact with it declaratively through NPC asset files.

### Standard Usage
The system invokes this builder during asset loading. A game designer or developer defines the sensor's usage in a data file, and the engine handles the instantiation.

```yaml
# Conceptual NPC Behavior Asset
# The "type: IsBusy" key is what causes the engine to
# find and use the BuilderSensorIsBusy factory.

stateMachine:
  initialState: Idle
  states:
    - name: Idle
      transitions:
        - to: Combat
          when:
            # This sensor will prevent the transition if the NPC is busy
            # with another high-priority task (e.g., a cinematic).
            - type: IsNot
              sensor:
                type: IsBusy
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorIsBusy()` in game logic. The asset pipeline is the sole owner of this class's lifecycle. Direct creation bypasses the entire data-driven AI framework.
-   **Extending for Gameplay Logic:** Do not subclass BuilderSensorIsBusy to add custom game logic. If a new sensor type is needed, create a new `BuilderSensor` and its corresponding `Sensor` implementation. This class is a sealed factory, not a base for extension.

## Data Pipeline
The BuilderSensorIsBusy is a single, critical step in the pipeline that transforms a static NPC definition into a live, thinking agent.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderSensorIsBusy** -> `SensorIsBusy` Object -> NPC State Machine -> Game State Evaluation

