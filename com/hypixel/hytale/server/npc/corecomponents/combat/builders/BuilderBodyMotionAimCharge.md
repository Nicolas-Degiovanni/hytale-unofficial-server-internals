---
description: Architectural reference for BuilderBodyMotionAimCharge
---

# BuilderBodyMotionAimCharge

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Configuration Builder

## Definition
```java
// Signature
public class BuilderBodyMotionAimCharge extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionAimCharge class is a data-driven factory within the server-side NPC behavior system. It is not a builder in the common fluent API sense; rather, it serves as a deserializer and constructor for a specific type of NPC action defined in external JSON asset files.

Its primary responsibility is to translate a static JSON configuration block into a live, executable `BodyMotionAimCharge` object. This class forms a critical bridge between the game's asset data and the runtime NPC AI. When an NPC's behavior asset specifies an "AimCharge" motion, an instance of this builder is used to parse its parameters, such as `RelativeTurnSpeed`.

A key architectural pattern demonstrated here is the use of a `DoubleHolder` for state. This defers the final resolution of a value until runtime by using an `ExecutionContext` provided via the `BuilderSupport` object. This allows a single static asset definition to produce behaviors that can be dynamically scaled by game logic, such as difficulty modifiers or character stats, without requiring separate asset files.

## Lifecycle & Ownership
- **Creation:** Instantiated transiently by a higher-level NPC asset parsing system when it encounters a JSON object corresponding to this behavior type. Developers should never instantiate this class directly.
- **Scope:** Short-lived. An instance's purpose is fulfilled once its `build` method is called. It exists only during the NPC's initialization or behavior-tree construction phase.
- **Destruction:** After the `build` method returns the configured `BodyMotionAimCharge` instance, the builder object has no further purpose and becomes eligible for garbage collection. It holds no persistent references and is not intended to be cached or reused for other configurations.

## Internal State & Concurrency
- **State:** The internal state, specifically the `relativeTurnSpeed` field, is mutable. It is populated exclusively through the `readConfig` method. After configuration, the object's state should be considered final for the subsequent `build` call.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single-threaded context, typically the server's main asset loading or entity initialization thread. Accessing an instance from multiple threads, especially concurrent calls to `readConfig`, will result in a corrupted state and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotion | O(1) | Constructs and returns a new `BodyMotionAimCharge` runtime instance using the previously parsed configuration. |
| readConfig(JsonElement) | BuilderBodyMotionAimCharge | O(N) | Parses the provided JSON, populating the builder's internal state. N is the number of properties in the JSON object. |
| getRelativeTurnSpeed(BuilderSupport) | double | O(1) | Resolves and returns the final turn speed value using the provided runtime context. |

## Integration Patterns

### Standard Usage
This class is used internally by the NPC asset loading pipeline. A developer defining NPC behavior in JSON will trigger its use implicitly. The conceptual flow is managed by the engine.

```java
// Conceptual example within an NPC Asset Factory

// 1. A JSON block for this motion is read from an asset file
JsonElement aimChargeJson = npcAsset.get("bodyMotionAimCharge");

// 2. The factory instantiates the builder and configures it
BuilderBodyMotionAimCharge builder = new BuilderBodyMotionAimCharge();
builder.readConfig(aimChargeJson);

// 3. During NPC spawning, the builder creates the runtime motion
BuilderSupport runtimeSupport = new BuilderSupport(npcExecutionContext);
BodyMotion motion = builder.build(runtimeSupport);

// 4. The final motion object is passed to the NPC's AI controller
npc.getBehaviorController().addMotion(motion);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderBodyMotionAimCharge()` in game logic. The class is part of the data-to-object pipeline and should not be used to programmatically define behavior. All configuration must originate from JSON assets.
- **State Mutation After Read:** Do not attempt to modify the builder's internal state after `readConfig` has been called. The object provides no public setters, and altering its state via reflection would violate its design contract.
- **Instance Re-use:** Do not use a single builder instance to process multiple, different JSON configurations. The `readConfig` method is not idempotent and does not clear previous state, which will lead to incorrect or merged configurations.

## Data Pipeline
The flow of data begins with a static asset file and ends with a runtime component attached to an NPC entity. This builder is a key transformation step in that pipeline.

> Flow:
> NPC Asset File (JSON) -> Engine JSON Parser -> **BuilderBodyMotionAimCharge**.readConfig() -> **BuilderBodyMotionAimCharge**.build() -> BodyMotionAimCharge (Runtime Instance) -> NPC Behavior Controller

