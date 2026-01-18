---
description: Architectural reference for SensorSwitch
---

# SensorSwitch

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorSwitch extends SensorBase {
```

## Architecture & Concepts
The SensorSwitch is a foundational component within the server-side NPC AI framework. It functions as a static, configuration-driven boolean gate. Unlike dynamic sensors that react to world state (e.g., player proximity, health status), the SensorSwitch's outcome is determined *once* at the time of the NPC's creation and remains constant throughout its lifecycle.

Its primary architectural role is to provide a low-cost, declarative way to enable or disable branches of an AI's behavior tree based on an NPC's asset definition. For example, it can be used to differentiate the logic for a "miniboss" variant of an enemy from a standard one by checking a flag in the asset file, without requiring complex runtime checks.

This component embodies the principle of separating static configuration from dynamic logic, ensuring that performance-critical AI decision-making can rely on a simple, pre-computed boolean check.

### Lifecycle & Ownership
- **Creation:** An instance of SensorSwitch is created by the server's NPC asset loading pipeline when an NPC entity is being constructed. The `BuilderSensorSwitch` reads the configuration from the NPC's definition file and uses it to instantiate this sensor. It is never created manually.
- **Scope:** The lifetime of a single NPC entity. Each NPC with this sensor defined in its AI configuration will possess its own unique, independent instance.
- **Destruction:** The object is marked for garbage collection when its owning NPC entity is destroyed and removed from the world. It has no explicit cleanup or teardown logic.

## Internal State & Concurrency
- **State:** Immutable. The core internal state is the `flag` field, which is marked as `final` and initialized in the constructor. Once an instance is created, its result is fixed and cannot be altered.
- **Thread Safety:** Conditionally thread-safe. The object itself contains no mutable state and poses no concurrency risk. However, it is designed to be operated on exclusively by the main server thread that manages NPC AI ticks. Passing its `matches` method world-state objects from other threads is an unsupported and dangerous operation.

## API Surface
The public API is minimal and focused entirely on evaluation within the AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor. Returns true only if the base sensor conditions pass and its internal, pre-configured flag is true. |
| getSensorInfo() | InfoProvider | O(1) | Always returns null. This sensor is designed as a simple gate and does not provide rich contextual data. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly from procedural code. It is exclusively configured declaratively within an NPC's asset files (e.g., JSON). The AI system, such as a Behavior Tree, then queries the sensor to control logical flow.

**Conceptual Asset Definition (YAML Example):**
```yaml
# In an NPC's asset definition file
ai:
  sensors:
    - name: IsEliteVariant
      type: SensorSwitch
      # The builder uses this value to set the internal flag
      switch: "npc.property('is_elite')"
  behaviors:
    - type: "Selector"
      children:
        - sequence:
            # This behavior only runs if the sensor passes
            condition: "sensor.IsEliteVariant.matches()"
            actions:
              - "UseEliteAbility"
        - sequence:
            # Fallback for non-elites
            actions:
              - "UseStandardAttack"
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorSwitch()`. The server's asset pipeline is the sole authority for its creation. Manual instantiation will lead to unmanaged objects that are not integrated into the NPC's AI controller.
- **Expecting Dynamic Behavior:** Do not poll this sensor expecting its value to change in response to game events. It is immutable. For dynamic checks, use a different sensor type that reads live component data.

## Data Pipeline
The data flow for SensorSwitch is centered on configuration and instantiation, not on a continuous stream of runtime data.

> Flow:
> NPC Asset File -> Asset Loading Service -> **BuilderSensorSwitch** -> **SensorSwitch** (Instance with baked `flag` value) -> AI Behavior Tree (Evaluation via `matches` call)

