---
description: Architectural reference for BuilderMotionControllerBase
---

# BuilderMotionControllerBase

**Package:** com.hypixel.hytale.server.npc.movement.controllers.builders
**Type:** Transient Factory Component

## Definition
```java
// Signature
public abstract class BuilderMotionControllerBase extends BuilderBaseWithType<MotionController> {
```

## Architecture & Concepts

The BuilderMotionControllerBase is an abstract class that serves as the foundational blueprint for creating runtime MotionController instances from configuration files. It is a critical component of the server-side NPC asset definition pipeline. Its primary role is to deserialize a JSON configuration block into a validated, in-memory representation that can later be used to instantiate and configure a concrete MotionController.

This class is not a runtime component and does not participate in the game simulation loop. Instead, it acts as a configuration-time factory, translating declarative JSON data into the parameters required by the NPC movement system.

A key architectural feature is its support for dynamic properties through wrappers like DoubleHolder and FloatHolder. This allows configuration values, such as movement speed, to be defined not just as static numbers but as expressions that are evaluated at runtime against an ExecutionContext. This provides significant flexibility for creating NPCs whose attributes can change based on game state, world properties, or other dynamic factors.

Subclasses of BuilderMotionControllerBase are responsible for implementing the final build logic and specifying the concrete MotionController class they produce, such as a walking, flying, or swimming controller.

## Lifecycle & Ownership

-   **Creation:** Instances are created by the NPC asset loading system, typically via reflection. When the system parses an NPC's JSON definition, it identifies the motion controller's "type" key and instantiates the corresponding builder registered under that name.
-   **Scope:** The lifecycle of a BuilderMotionControllerBase instance is extremely short and confined to the asset loading and validation phase. It exists only to parse its configuration block and serve as a template for the final runtime object.
-   **Destruction:** Once the corresponding MotionController instance has been built and configured, the builder object is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable. Its fields are populated sequentially as the `readCommonConfig` method processes a JSON object. The state is effectively a cache of the deserialized configuration values.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated, configured, and used within the single-threaded context of the asset loading pipeline. Accessing an instance from multiple threads, especially during configuration parsing, will result in a corrupted state and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement) | Builder<MotionController> | O(N) | Deserializes the provided JSON, populating the builder's internal state. N is the number of keys. |
| validate(...) | boolean | O(1) | Executes validation logic against the parsed state as part of the global asset validation pipeline. |
| getIdentifier() | String | O(1) | Retrieves the unique key name under which this builder type is registered in the NPCPlugin. |
| getMaxHorizontalSpeed(BuilderSupport) | double | O(E) | Evaluates and returns the maximum horizontal speed. Complexity depends on the underlying expression E. |
| getMaxHeadRotationSpeed(BuilderSupport) | float | O(E) | Evaluates and returns the maximum head rotation speed in radians. |
| getClassType() | Class<? extends MotionController> | O(1) | Abstract method. Subclasses must implement this to return the concrete MotionController class they build. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by game logic developers. It is invoked internally by the asset loading framework. The conceptual flow is as follows:

```java
// Conceptual example of the asset loader's logic
String controllerType = json.get("type").getAsString();
BuilderMotionControllerBase builder = NPCPlugin.get().getMotionControllerBuilder(controllerType);

// The framework populates the builder from the JSON definition
builder.readCommonConfig(json);

// The builder is then used to create the runtime instance
// Note: The actual build method is part of the broader builder pattern
MotionController runtimeController = builder.build(executionContext);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BuilderMotionControllerBase()` or its subclasses. The asset system relies on a central registry to look up and instantiate the correct builder based on the "type" key in JSON.
-   **State Reuse:** Do not cache and reuse a builder instance to configure multiple MotionController objects. Each builder is a single-use object tied to a specific JSON definition block.
-   **Runtime Access:** Do not attempt to retrieve or reference a builder from the main game loop or from an active NPC. The builder's lifecycle is finished before the NPC is ever spawned. All necessary data is transferred to the runtime MotionController instance.

## Data Pipeline

The BuilderMotionControllerBase is a transformation step in the data pipeline that converts static asset definitions into live game objects.

> Flow:
> NPC Asset JSON File -> Gson Parser -> `JsonElement` -> **BuilderMotionControllerBase.readCommonConfig** -> In-Memory Builder State -> Build Process -> `MotionController` Runtime Instance

