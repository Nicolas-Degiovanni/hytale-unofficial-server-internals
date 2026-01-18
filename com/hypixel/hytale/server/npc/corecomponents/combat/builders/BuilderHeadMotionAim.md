---
description: Architectural reference for BuilderHeadMotionAim
---

# BuilderHeadMotionAim

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderHeadMotionAim extends BuilderHeadMotionBase {
```

## Architecture & Concepts
The BuilderHeadMotionAim class is a configuration and factory component within the server-side NPC asset system. It operates exclusively during the asset loading phase, translating declarative NPC behavior definitions from JSON into concrete, runtime game components.

This class embodies the **Builder Pattern**. Its sole responsibility is to parse a specific JSON structure that defines an NPC's aiming behavior and use that data to construct an immutable HeadMotionAim instance. The resulting HeadMotionAim object is then attached to an NPC entity and used by the combat AI to control how the NPC's head tracks and aims at targets.

BuilderHeadMotionAim acts as a critical bridge between the static data defined by game designers and the dynamic, in-game object model. It isolates the complexity of JSON parsing, validation, and default value application from the runtime HeadMotionAim component, which is concerned only with executing the aiming logic.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's NPC asset pipeline when it encounters a "HeadMotionAim" type during the deserialization of an NPC's component list from a JSON file. It is never created directly by gameplay logic.
- **Scope:** Ephemeral and short-lived. Its lifecycle is strictly confined to the asset loading process. It exists only to configure and construct a single HeadMotionAim instance.
- **Destruction:** The builder object becomes eligible for garbage collection immediately after the `build` method is invoked and the resulting HeadMotionAim component is passed to the parent NPC factory. It does not persist into the active game state.

## Internal State & Concurrency
- **State:** Highly mutable. The class is designed to accumulate configuration state from a JSON source via the `readConfig` method. This state includes parameters like `spread`, `hitProbability`, and `deflection`. The `relativeTurnSpeed` field uses a DoubleHolder, indicating it may represent a dynamic value resolved later using the ExecutionContext.

- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the asset loading pipeline. The internal state is unprotected, and concurrent calls to `readConfig` or other methods will lead to race conditions and unpredictable component behavior. The object's state must be considered volatile until the `build` method is called.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | HeadMotionAim | O(1) | Constructs and returns a new HeadMotionAim runtime component. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderHeadMotionAim | O(N) | Populates the builder's state from a JSON object, where N is the number of keys. Validates inputs and applies defaults. |
| getRelativeTurnSpeed(BuilderSupport) | double | O(1) | Retrieves the configured turn speed. Requires a BuilderSupport context, as the value may be resolved dynamically. |

## Integration Patterns

### Standard Usage
The BuilderHeadMotionAim is used exclusively by the internal NPC asset loading system. The conceptual flow involves creating a new builder, configuring it from a JSON source, and then building the final runtime component.

```java
// Conceptual example within the asset loading system
JsonElement aimConfig = npcDefinition.get("headMotionAim");
BuilderSupport support = ...; // Provided by the asset system

BuilderHeadMotionAim builder = new BuilderHeadMotionAim();
builder.readConfig(aimConfig);

// The final, immutable runtime component is created
HeadMotionAim runtimeComponent = builder.build(support);

// The builder is now discarded
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a builder instance to create multiple HeadMotionAim components. Each component requires a new, clean builder instance to ensure configuration isolation and prevent state leakage.
- **Manual Instantiation:** Gameplay logic should never instantiate this class using `new`. The NPC asset system is solely responsible for its creation and lifecycle management.
- **Incomplete Initialization:** Invoking `build` before `readConfig` has been called will result in a component configured with default-only parameters, which will likely produce unintended aiming behavior.

## Data Pipeline
The class functions as a transformation step in the data pipeline that converts static asset files into live game objects.

> Flow:
> JSON Asset File -> NPC Asset Parser -> **BuilderHeadMotionAim.readConfig()** -> **BuilderHeadMotionAim.build()** -> HeadMotionAim Instance -> NPC Entity Component System

