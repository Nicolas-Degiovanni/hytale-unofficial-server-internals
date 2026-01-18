---
description: Architectural reference for BuilderActionOverrideAltitude
---

# BuilderActionOverrideAltitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionOverrideAltitude extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionOverrideAltitude class is a factory component within the server-side NPC Behavior system. Its sole responsibility is to translate a JSON configuration definition into a concrete, executable ActionOverrideAltitude object.

This class is a fundamental part of Hytale's data-driven design for AI. It allows game designers to define complex NPC behaviors in external JSON assets without altering core Java code. Each `Builder` class, like this one, corresponds to a specific instruction an NPC can execute.

Architecturally, it serves as a validation and instantiation gateway. It consumes raw configuration data, validates it against engine constraints (e.g., altitude must be a positive, monotonic range), and produces a ready-to-use Action object that can be integrated into an NPC's behavior tree. It decouples the high-level AI design (JSON) from the low-level engine implementation (Java).

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the server's asset loading pipeline when it encounters the corresponding action type in an NPC behavior asset file. It is never created manually by game logic code.
-   **Scope:** Extremely short-lived and transient. An instance of this builder exists only for the duration of parsing a single JSON entry and building one Action object from it.
-   **Destruction:** The builder instance is immediately eligible for garbage collection after the `build` method has been called and its resulting Action has been returned to the asset loader. It does not persist.

## Internal State & Concurrency
-   **State:** This object is stateful. It maintains the parsed `desiredAltitudeRange` as internal mutable state between the `readConfig` and `build` calls. This state is isolated to the single Action being constructed.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The server's asset loading systems guarantee that each builder instance is created, configured, and used within a single, synchronized thread of execution.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionOverrideAltitude instance using the previously parsed configuration. |
| readConfig(JsonElement) | Builder<Action> | O(N) | Parses the input JSON, validates the `DesiredAltitudeRange` field, and stores the result internally. N is the size of the JSON data. |
| getDesiredAltitudeRange(BuilderSupport) | double[] | O(1) | Retrieves the configured altitude range. Requires a support context to resolve potentially dynamic values. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in standard game logic. The following example illustrates the internal pattern used by the asset loading system to process a configuration and produce an action.

```java
// System-level code within the asset loader
BuilderActionOverrideAltitude builder = new BuilderActionOverrideAltitude();

// 1. Configure the builder from a JSON source
builder.readConfig(npcActionJsonData);

// 2. Build the final, immutable Action object
// The builder instance is now discarded
Action action = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Never reuse a builder instance to create multiple Action objects. Each builder is stateful and designed for a single `readConfig` -> `build` lifecycle.
-   **Calling Build Before Read:** Do not call the `build` method before `readConfig` has been successfully invoked. This will result in an unconfigured and invalid Action that will cause runtime errors in the AI system.
-   **Manual Instantiation:** Game logic should never instantiate this class with `new`. NPC behaviors are defined entirely within data assets.

## Data Pipeline
The primary flow involves the transformation of declarative configuration data into an executable engine object.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionOverrideAltitude.readConfig()** -> In-Memory State -> **BuilderActionOverrideAltitude.build()** -> ActionOverrideAltitude Instance -> NPC Behavior Tree

