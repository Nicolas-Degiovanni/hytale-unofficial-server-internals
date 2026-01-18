---
description: Architectural reference for BuilderBodyMotionLand
---

# BuilderBodyMotionLand

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderBodyMotionLand extends BuilderBodyMotionFind {
```

## Architecture & Concepts
The BuilderBodyMotionLand class is a specialized factory component within the server-side NPC asset system. Its primary function is to parse a JSON configuration block and construct an immutable, runtime-ready BodyMotionLand behavior object.

This class embodies the **Builder Pattern** for a specific NPC movement strategy: landing. It acts as a translation layer, converting declarative data from a design file (JSON) into an executable Java object. It is part of a larger framework where different NPC behaviors are defined by distinct builder classes, allowing for a highly extensible and data-driven AI system.

Architecturally, it sits at the asset loading boundary. It is consumed by a higher-level asset parser that iterates through an NPC's behavior definitions. Its existence is ephemeral, serving only to configure and build its corresponding runtime component. The validation logic within this class is critical, as it enforces design-time constraints, preventing invalid NPC configurations from ever reaching the live game server.

### Lifecycle & Ownership
- **Creation:** Instantiated by the central NPC asset parsing system when it encounters a behavior definition of type BodyMotionLand in an NPC's JSON configuration file.
- **Scope:** Ephemeral and short-lived. Its scope is confined to the asset loading and validation phase for a single NPC type. It does not persist into the game runtime.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method is called and the resulting BodyMotionLand instance is registered with the parent NPC asset. It holds no references that would prolong its life.

## Internal State & Concurrency
- **State:** The class maintains a mutable internal state during configuration. Its primary state field is `goalLenience`, which defines the acceptable distance from a target for a landing to be considered successful. This state is populated exclusively by the `readConfig` method. Once the `build` method is called, the state is effectively frozen and transferred to the new BodyMotionLand object.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the single-threaded context of the NPC asset loading pipeline. Concurrent access, particularly to the `readConfig` method, will result in a corrupted and unpredictable configuration state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionLand | O(1) | Constructs and returns the final, immutable BodyMotionLand runtime object. |
| readConfig(JsonElement) | BuilderBodyMotionLand | O(N) | Parses the provided JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| validate(...) | boolean | O(1) | Enforces configuration contracts. Critically, it requires the NPC to have both MotionControllerFly and MotionControllerWalk capabilities. |
| getGoalLenience(BuilderSupport) | double | O(1) | Resolves and returns the configured goal lenience value. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by game logic developers. It is invoked automatically by the asset loading framework. The conceptual flow is as follows.

```java
// Conceptual example of framework usage
BuilderBodyMotionLand builder = new BuilderBodyMotionLand();

// The framework populates the builder from a JSON source
builder.readConfig(npcJson.get("behaviorConfig"));

// The framework validates the configuration in context
boolean isValid = builder.validate(configName, validationHelper, ...);
if (!isValid) {
    throw new AssetLoadException("Invalid BodyMotionLand config");
}

// The final runtime object is created and stored
BodyMotionLand runtimeBehavior = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Never instantiate this class directly in game systems. The NPC asset pipeline is the sole owner and creator of this builder.
- **State Re-use:** Do not hold a reference to this builder after calling `build`. Its lifecycle is complete at that point, and its state should be considered stale.
- **Skipping Validation:** Bypassing the `validate` method can lead to severe runtime errors. The validation step ensures that the NPC has the required motion controllers (e.g., wings and legs) to perform a landing maneuver, preventing runtime ClassCastExceptions or NullPointerExceptions.

## Data Pipeline
The primary role of this class is to transform declarative configuration data into an executable object.

> Flow:
> NPC Definition (JSON) -> Asset Parser -> **BuilderBodyMotionLand.readConfig()** -> In-Memory State -> **BuilderBodyMotionLand.build()** -> BodyMotionLand (Runtime Object) -> NPC Behavior Controller

