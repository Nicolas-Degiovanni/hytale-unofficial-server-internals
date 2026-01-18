---
description: Architectural reference for BuilderEntityFilterHeightDifference
---

# BuilderEntityFilterHeightDifference

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient Configuration Builder

## Definition
```java
// Signature
public class BuilderEntityFilterHeightDifference extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterHeightDifference is a key component within the server's data-driven NPC (Non-Player Character) asset system. Its primary function is to act as a factory, translating a declarative JSON configuration block into a concrete, executable Java object.

This class is not part of the real-time game loop. Instead, it operates during the server's asset loading phase. When the server parses an NPC definition file, it uses builders like this one to construct the various components that define the NPC's behavior. Specifically, this builder is responsible for creating an IEntityFilter that can test whether another entity is within a specified vertical distance (height difference) from the NPC.

This pattern decouples game logic from the source code, allowing designers and developers to define complex NPC behaviors, such as targeting conditions, in external JSON files without recompiling the server. This builder is the bridge between the static data definition and the live, in-game filtering logic.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the core NPC asset loading system, typically via a registry that maps a JSON type name (e.g., "EntityFilterHeightDifference") to this builder class. It is never created directly by game logic developers.
-   **Scope:** Extremely short-lived and transient. An instance exists only for the duration required to parse a single JSON object and build one IEntityFilter instance.
-   **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method is called and its result is passed to the parent component. It holds no persistent state and is not retained.

## Internal State & Concurrency
-   **State:** This class is stateful and highly mutable. Its purpose is to accumulate configuration from the `readConfig` method into its internal fields, such as `useEyePosition` and `heightDifference`. These fields are wrapped in Holder objects, which may allow for deferred or context-sensitive value resolution. The state represents the complete configuration for a single `EntityFilterHeightDifference` instance.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be used in a single-threaded context during asset loading. Invoking its methods, particularly `readConfig`, from multiple threads on the same instance will result in a corrupted state and unpredictable behavior. Each configuration block must be processed by a new, dedicated builder instance.

## API Surface
The public API is designed for use by the asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterHeightDifference instance. |
| readConfig(JsonElement) | Builder | O(N) | Parses the input JSON, populates internal state. N is the number of keys. |
| getHeightDifference(BuilderSupport) | double[] | O(1) | Retrieves the configured height range. Requires a support context. |
| isUseEyePosition(BuilderSupport) | boolean | O(1) | Retrieves the configured flag for using eye position. Requires a support context. |

## Integration Patterns

### Standard Usage
This class is used internally by the asset system. A developer would define the filter in a JSON file, and the system would use the builder behind the scenes.

**Example NPC Definition (JSON):**
```json
{
  "type": "EntityFilterHeightDifference",
  "config": {
    "HeightDifference": [-1.0, 5.0],
    "UseEyePosition": true
  }
}
```

**Internal System Usage (Conceptual):**
```java
// The asset system finds the correct builder for the "type"
BuilderEntityFilterHeightDifference builder = new BuilderEntityFilterHeightDifference();

// It passes the "config" object to the builder
JsonElement configData = parseJson(jsonFile).get("config");
builder.readConfig(configData);

// It then builds the final, usable filter object
BuilderSupport support = ...; // The current asset loading context
IEntityFilter filter = builder.build(support);

// This 'filter' instance is then attached to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Manual Configuration:** Do not manually instantiate this class with `new` and attempt to configure it via setters. Its design is coupled to the `readConfig` flow from a JSON source.
-   **Instance Re-use:** Do not reuse a single builder instance to parse multiple, different JSON configurations. The internal state will be overwritten, leading to incorrect results. Create a new builder for each distinct filter definition.
-   **Calling `build` before `readConfig`:** This will produce a filter with default values, which is almost certainly not the intended behavior. The `readConfig` method is a mandatory step in the lifecycle.

## Data Pipeline
This builder is a critical step in the pipeline that transforms static data into executable server logic.

> Flow:
> NPC Definition (*.json file*) -> Asset Loading Service -> JSON Parser -> **BuilderEntityFilterHeightDifference** -> `EntityFilterHeightDifference` (Instance) -> NPC Behavior Tree / Sensor System

