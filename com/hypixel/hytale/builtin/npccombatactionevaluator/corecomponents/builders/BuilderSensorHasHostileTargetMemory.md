---
description: Architectural reference for BuilderSensorHasHostileTargetMemory
---

# BuilderSensorHasHostileTargetMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorHasHostileTargetMemory extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorHasHostileTargetMemory class is a factory component within the server-side NPC AI framework. It does not represent the AI logic itself, but rather the *recipe* for constructing that logic. Its sole responsibility is to instantiate a runtime SensorHasHostileTargetMemory object.

This class acts as a bridge between the static data defined in NPC asset files (e.g., JSON or HOCON configurations) and the live, executable objects that form an NPC's behavior tree. By separating the *builder* from the *product* (the Sensor), the engine can manage a clean asset pipeline where descriptive, serializable builder objects are used to construct complex, stateful runtime components.

This component is typically referenced by name within a behavior tree definition file. The asset loading system resolves this name to the builder class, instantiates it, and invokes its build method to produce the final Sensor object that gets attached to an NPC's AI controller.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's NPC asset loading system when parsing an NPC behavior definition. This is an automated process driven by asset configuration, not manual code invocation.
-   **Scope:** Extremely short-lived and transient. An instance of this builder exists only for the brief moment required to construct the corresponding Sensor object during the NPC's initialization phase.
-   **Destruction:** The builder object becomes eligible for garbage collection immediately after its build method has been called and the resulting Sensor has been integrated into the NPC's behavior tree. It does not persist.

## Internal State & Concurrency
-   **State:** Stateless and immutable. This class contains no mutable fields. Its behavior is entirely defined by its type and hardcoded methods, such as getShortDescription.
-   **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, it can be safely used by multiple threads during parallel asset loading without risk of race conditions or data corruption.

## API Surface
The public API is minimal, reflecting its singular purpose as a factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorHasHostileTargetMemory instance. This is the primary factory method. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for tooling and editors. |
| getLongDescription() | String | O(1) | Provides a detailed description for tooling and editors. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns Stable, indicating this is a production-ready component. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in gameplay code. Instead, it is declared within an NPC's behavior asset file. The engine's asset pipeline handles its instantiation and usage.

A conceptual asset definition might look like this:

```yaml
# Example NPC Behavior Asset (conceptual)
behaviorTree:
  root:
    type: "Selector"
    children:
      - type: "Sequence"
        children:
          - sensor: "HasHostileTargetMemory" # This string resolves to BuilderSensorHasHostileTargetMemory
            # ... other properties
          - action: "AttackMelee"
            # ... other properties
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorHasHostileTargetMemory()` in game logic. The asset system manages the lifecycle. Manually creating builders bypasses the entire asset-driven AI configuration pipeline.
-   **Manual Invocation:** Do not acquire an instance of this builder to call its build method. The NPC's behavior tree constructor is the sole intended caller.

## Data Pipeline
This component sits at a critical juncture in the asset-to-runtime pipeline for NPC AI.

> Flow:
> NPC Behavior Asset File (JSON/HOCON) -> Server Asset Loader -> **BuilderSensorHasHostileTargetMemory** (Instantiation) -> `build()` Invocation -> SensorHasHostileTargetMemory (Runtime Object) -> NPC Behavior Tree Integration

