---
description: Architectural reference for BuilderSensorCombatActionEvaluator
---

# BuilderSensorCombatActionEvaluator

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorCombatActionEvaluator extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorCombatActionEvaluator is a factory class within the server's NPC asset definition pipeline. Its sole responsibility is to parse a JSON configuration block and construct an instance of SensorCombatActionEvaluator. It acts as the critical translation layer between static NPC definition files (JSON) and live, in-game AI sensor objects.

In Hytale's NPC AI framework, a *Sensor* is a component that evaluates world conditions to trigger behaviors or state transitions. This specific builder configures a sensor that is fundamental to dynamic combat. It funnels high-level decisions from the main Combat Action Evaluator (CAE) system into the NPC's state machine.

The primary function of the resulting sensor is to check if the NPC's current target is within the desired attack range, a value determined by the CAE. This builder configures how the sensor reads this desired range and other combat parameters (e.g., positioning angle) from a shared memory space known as the ValueStore.

### Lifecycle & Ownership
-   **Creation:** Instantiated reflectively by the NPC asset loading system when it encounters the corresponding sensor type in an NPC's JSON definition file. The `readConfig` method is immediately called to populate the builder's state from the JSON data.
-   **Scope:** The builder object is extremely short-lived. It exists only during the parsing and validation phase of a single NPC asset.
-   **Destruction:** The builder is eligible for garbage collection as soon as its `build` method has been called and the resulting SensorCombatActionEvaluator object has been integrated into the NPC's runtime asset. It does not persist into the game session.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** during configuration. Fields like targetInRange and the various ValueStore slot functions are populated by the `readConfig` method. After this initial parsing, the state is effectively immutable for the remainder of the builder's brief lifecycle. The use of Holder classes like BooleanHolder and DoubleHolder allows for deferred value resolution, but the configuration itself does not change after parsing.
-   **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. It is intended to be used exclusively within the single-threaded context of the NPC asset loading pipeline. All method calls are expected to occur in a strict, sequential order: constructor, `readConfig`, `validate`, and finally `build`.

## API Surface
The public API is designed to be used by the server's asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns the configured SensorCombatActionEvaluator instance. |
| readConfig(JsonElement) | BuilderSensorCombatActionEvaluator | O(N) | Parses the JSON configuration and populates the builder's internal state. N is the number of keys. |
| validate(...) | boolean | O(1) | Verifies that the configuration is semantically valid, such as ensuring the sensor is part of a state. |
| isTargetInRange(BuilderSupport) | boolean | O(1) | Accessor for the configured `TargetInRange` boolean flag. |
| getMinRangeStoreSlot(BuilderSupport) | int | O(1) | Returns the ValueStore slot index for the minimum desired attack range. |
| getMaxRangeStoreSlot(BuilderSupport) | int | O(1) | Returns the ValueStore slot index for the maximum desired attack range. |
| getPositioningAngleStoreSlot(BuilderSupport) | int | O(1) | Returns the ValueStore slot index for the desired positioning angle relative to the target. |

## Integration Patterns

### Standard Usage
This class is not used directly by developers. It is invoked by the engine's asset loader. The developer's interaction is purely through the NPC's JSON definition file.

A conceptual view of the engine's interaction:

```java
// Engine Pseudo-Code during asset loading
JsonElement sensorJson = findSensorDefinitionInNpcFile();
BuilderSensorCombatActionEvaluator builder = new BuilderSensorCombatActionEvaluator();

// 1. Configure the builder from JSON
builder.readConfig(sensorJson);

// 2. Validate the configuration
List<String> errors = new ArrayList<>();
builder.validate("MyNpc", validationHelper, context, scope, errors);

// 3. Construct the final sensor object
if (errors.isEmpty()) {
    Sensor liveSensor = builder.build(builderSupport);
    npcStateMachine.addSensor(liveSensor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorCombatActionEvaluator()` in game logic. The object is useless without being configured via `readConfig` from a valid JSON source.
-   **Calling `build` before `readConfig`:** Attempting to build the sensor before parsing a configuration will result in a misconfigured object that will cause runtime exceptions or undefined AI behavior.
-   **State Re-use:** Do not attempt to re-use a single builder instance to configure multiple sensors. They are designed as single-use, throwaway objects.

## Data Pipeline
This builder operates within the asset *configuration* pipeline. It transforms static data into a runtime object.

> **Configuration Flow:**
> NPC JSON File -> Server Asset Loader -> **BuilderSensorCombatActionEvaluator.readConfig()** -> Internal State Population -> **BuilderSensorCombatActionEvaluator.build()** -> Live SensorCombatActionEvaluator Object

The *product* of this builder, the SensorCombatActionEvaluator, then participates in the live *runtime* data pipeline for NPC combat AI.

> **Runtime Flow (of the created Sensor):**
> Combat Action Evaluator (writes to ValueStore) -> ValueStore -> **SensorCombatActionEvaluator** (reads range from ValueStore) -> Sensor Activation -> NPC State Machine Transition

