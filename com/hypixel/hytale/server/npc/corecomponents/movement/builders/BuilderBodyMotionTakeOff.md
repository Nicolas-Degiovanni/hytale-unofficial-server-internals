---
description: Architectural reference for BuilderBodyMotionTakeOff
---

# BuilderBodyMotionTakeOff

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionTakeOff extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionTakeOff class is a factory component within the server-side NPC asset system. Its sole responsibility is to parse a JSON configuration snippet and construct a corresponding BodyMotionTakeOff object. This pattern decouples the data-driven definition of an NPC's behavior from its runtime implementation.

This builder specifically defines the parameters for an NPC to transition from a ground-based movement state to a flying state. It acts as a configuration bridge, ensuring that the necessary motion controllers—MotionControllerWalk and MotionControllerFly—are available on the NPC before allowing this behavior to be attached. The builder's lifecycle is ephemeral, existing only during the initial asset loading and validation phase.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level asset parsing system when it encounters a JSON object specifying a "take off" body motion. This is not intended for manual instantiation by developers.
- **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single JSON configuration, validating it, and building the final BodyMotionTakeOff object.
- **Destruction:** The builder instance is immediately eligible for garbage collection after the `build` method returns its product. It holds no persistent references and is not registered in any service locator.

## Internal State & Concurrency
- **State:** The class is stateful but temporarily so. It maintains a mutable `jumpSpeed` field which is populated by the `readConfig` method. This internal state is used exclusively to parameterize the BodyMotionTakeOff object during its construction.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used in a single-threaded context during the NPC asset loading pipeline. The sequence of `readConfig` followed by `build` is inherently state-dependent and must not be accessed concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotion | O(1) | Constructs and returns the final BodyMotionTakeOff instance. |
| readConfig(JsonElement) | Builder<BodyMotion> | O(1) | Parses the JSON data to configure the builder's internal state, such as `jumpSpeed`. |
| validate(...) | boolean | O(1) | Verifies that the NPC asset has the required MotionControllerWalk and MotionControllerFly components. |
| getJumpSpeed() | double | O(1) | Returns the configured jump speed. Primarily used by the BodyMotionTakeOff constructor. |

## Integration Patterns

### Standard Usage
This class is used internally by the NPC asset loading framework. A developer would define the behavior in a JSON file, not interact with the builder in Java code. The system's internal usage pattern is as follows.

```java
// System-level code (conceptual)
JsonElement motionConfig = parseNpcAssetFile(".../npc.json");

BuilderBodyMotionTakeOff builder = new BuilderBodyMotionTakeOff();
builder.readConfig(motionConfig);

// Validation is a critical step before building
boolean isValid = builder.validate(...);
if (isValid) {
    BodyMotion takeOffMotion = builder.build(builderSupport);
    npc.addBehavior(takeOffMotion);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Configuration:** Do not manually instantiate this builder and attempt to configure it programmatically. Its design is coupled to the JSON-driven asset pipeline.
- **State Reuse:** Do not reuse a builder instance to build multiple BodyMotion objects. Each instance is meant to process a single configuration and should be discarded after use.
- **Skipping Validation:** The `validate` method is a critical guard. Bypassing it can lead to runtime exceptions or undefined behavior if an NPC is instructed to take off without having the necessary motion controllers (e.g., MotionControllerFly).

## Data Pipeline
The flow of data from configuration to a runtime object is linear and unidirectional.

> Flow:
> NPC JSON Asset -> Asset Loading Service -> **BuilderBodyMotionTakeOff** -> `BodyMotionTakeOff` Instance -> NPC Behavior Controller

