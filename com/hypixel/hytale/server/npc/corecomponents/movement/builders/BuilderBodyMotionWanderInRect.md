---
description: Architectural reference for BuilderBodyMotionWanderInRect
---

# BuilderBodyMotionWanderInRect

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionWanderInRect extends BuilderBodyMotionWanderBase {
```

## Architecture & Concepts
The BuilderBodyMotionWanderInRect class is a data-driven factory component within the server's NPC asset loading pipeline. It adheres to the Builder design pattern, specifically tailored for deserializing a configuration block from a JSON asset file into a concrete runtime object, BodyMotionWanderInRect.

Its primary responsibility is to translate a declarative JSON definition for an NPC's rectangular wandering behavior into an executable movement component. It extends BuilderBodyMotionWanderBase, inheriting the logic for common "wander" parameters, while this subclass is responsible for parsing and validating the unique geometric constraints of the rectangle: its width and depth.

This class acts as a transient intermediary. It is instantiated by the asset system, configured with a specific JSON element, and then consumed to produce a single, immutable BodyMotionWanderInRect instance. It is not intended for long-term storage or reuse.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by a higher-level asset factory, such as an NpcComponentBuilderRegistry. This registry maps a `type` string from the NPC's JSON definition to this specific builder class. A developer will never instantiate this class directly.
-   **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single movement component within an NPC asset file.
-   **Destruction:** The instance becomes eligible for garbage collection immediately after the `build` method is called and its result is attached to the NPC being constructed. It holds no external references and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state is **Mutable**. The `width` and `depth` fields are populated by the `readConfig` method. This state is temporary and serves only to accumulate configuration before the final object is constructed. If the JSON file omits these fields, they are populated with hardcoded default values (10.0).
-   **Thread Safety:** This class is **Not Thread-Safe** and must not be shared across threads. It is designed to be used exclusively within a single-threaded asset loading context. Concurrent calls to `readConfig` on the same instance would result in a race condition, corrupting the configuration state.

## API Surface
The public API is designed for a single, linear workflow: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderBodyMotionWanderInRect | O(1) | Parses the provided JSON, populating internal state for width and depth. Validates that values are greater than zero. Returns `this` for method chaining. |
| build(BuilderSupport support) | BodyMotionWanderInRect | O(1) | Terminal operation. Consumes the internal state to construct and return a new BodyMotionWanderInRect instance. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary of the behavior. Used for tooling and diagnostics. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description of the behavior. Used for tooling and diagnostics. |

## Integration Patterns

### Standard Usage
This class is not used directly by feature developers but is invoked by the core asset loading system. The pattern is to instantiate, configure from JSON, and immediately build the final component.

```java
// Hypothetical usage within an asset loading service

// 1. A JSON object for this component is retrieved
JsonElement motionConfig = npcAssetJson.get("movement");

// 2. The appropriate builder is instantiated by a factory
BuilderBodyMotionWanderInRect builder = new BuilderBodyMotionWanderInRect();

// 3. The builder is configured from the JSON data
builder.readConfig(motionConfig);

// 4. The final runtime component is built
// BuilderSupport provides access to shared resources or context
BodyMotionWanderInRect runtimeComponent = builder.build(builderSupport);

// 5. The component is attached to the NPC
npc.addMovementComponent(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Manual Configuration:** Do not manually instantiate and configure this builder in game logic. Its purpose is to deserialize data, not to be configured programmatically.
-   **Reusing Builder Instances:** A builder instance must not be reused. It is stateful and designed for a single build operation. Using it to build a second component will result in an object with identical configuration to the first.
-   **Calling build Before readConfig:** Invoking `build` on a newly created instance without calling `readConfig` will produce a component with uninitialized or default state (width and depth of 0.0 from the JVM, not the 10.0 default from `readConfig`), which will likely cause unpredictable runtime behavior.

## Data Pipeline
This builder is a key stage in the pipeline that transforms a static data asset into a live server object.

> Flow:
> NPC JSON Asset File -> Server Asset Manager -> JSON Parser -> **BuilderBodyMotionWanderInRect** (`readConfig` -> `build`) -> BodyMotionWanderInRect (Runtime Component) -> NPC Entity State

