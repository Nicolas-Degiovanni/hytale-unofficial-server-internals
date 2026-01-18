---
description: Architectural reference for BuilderSensorDamage
---

# BuilderSensorDamage

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorDamage extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorDamage class is a factory component within the NPC asset loading pipeline. Its primary responsibility is to translate a declarative JSON configuration into a concrete, runtime instance of a SensorDamage object. It acts as a bridge between the static data definitions of an NPC and the live, in-game behavior system.

This builder is a configuration-time object. It is instantiated, configured via the readConfig method, and then used once to produce a SensorDamage object, which is the actual runtime component that checks if an NPC has taken damage.

A key architectural feature is its built-in validation logic. Methods like validateAny and validateBooleanImplicationAnyAntecedent are called within readConfig to enforce strict configuration rules at load time. This prevents invalid or logically inconsistent NPC behaviors from ever being instantiated, shifting error detection from unpredictable runtime failures to a deterministic asset loading phase.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's NPC asset loading framework. When the framework parses an NPC definition file and encounters a sensor of this type, it creates a new BuilderSensorDamage instance to handle the configuration.
- **Scope:** Ephemeral and extremely short-lived. An instance exists only for the duration of parsing a single sensor definition from a JSON object.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the build method is called and the resulting SensorDamage object is returned to the asset loader. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state consists of several boolean flags (e.g., combatDamage, drowningDamage) and a String (targetSlot). This state is highly **mutable**, as its sole purpose is to be populated by the readConfig method. The state is a temporary container for parameters that will be passed to the SensorDamage constructor.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during asset loading. Accessing a single instance from multiple threads will lead to race conditions and a corrupted final object. The asset loading system must guarantee that a new builder is created for each sensor definition it processes.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorDamage | O(1) | Constructs the final SensorDamage runtime object using the currently configured state. Throws exceptions if the BuilderSupport context is invalid. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the JSON definition, populates internal state, and executes validation rules. N is the number of keys in the JSON object. Throws exceptions on invalid configuration. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its integer ID via the provided BuilderSupport context. Returns Integer.MIN_VALUE if no slot is defined. |

## Integration Patterns

### Standard Usage
The builder is intended to be used exclusively by the internal asset loading system. A new instance is created, configured from a JSON source, and then used to build the final sensor object, which is then integrated into an NPC's behavior tree.

```java
// Conceptual usage within the asset loading system
BuilderSupport supportContext = ...;
JsonElement sensorJsonDefinition = ...;

BuilderSensorDamage builder = new BuilderSensorDamage();
builder.readConfig(sensorJsonDefinition);
SensorDamage sensor = builder.build(supportContext);

// The 'sensor' object is now a fully configured, runtime-ready component
// that can be attached to an NPC's behavior state machine.
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not attempt to reuse a BuilderSensorDamage instance to parse a second JSON definition. The internal state is not reset between calls to readConfig, which will result in a corrupted and unpredictable configuration.
- **Direct Field Manipulation:** Avoid setting fields like combatDamage or targetSlot directly. Bypassing the readConfig method will also bypass the critical validation logic, potentially creating an invalid SensorDamage object that will cause runtime errors.
- **Manual Instantiation:** Application code should never instantiate this class directly using new. The NPC behavior system relies on its own factories and asset loading pipeline to manage the creation and configuration of these components.

## Data Pipeline
The BuilderSensorDamage is a key transformation step in the data pipeline that converts static NPC assets into live game objects.

> Flow:
> NPC Definition (JSON File) -> Asset Loading Service -> **BuilderSensorDamage.readConfig()** -> **BuilderSensorDamage.build()** -> SensorDamage (Runtime Object) -> NPC Behavior Tree

