---
description: Architectural reference for BuilderHeadMotionObserve
---

# BuilderHeadMotionObserve

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderHeadMotionObserve extends BuilderHeadMotionBase {
```

## Architecture & Concepts
The BuilderHeadMotionObserve class is a key component of the server-side NPC asset pipeline. It operates as a configuration-driven factory, translating a JSON definition into a concrete, runtime behavior object: HeadMotionObserve.

Its primary architectural role is to act as a bridge between the static data representation of an NPC's behavior (defined in JSON files) and the live, in-game object that executes that behavior. It encapsulates the logic for parsing, validating, and setting default values for the "Observe" head motion pattern.

This class follows the **Builder Pattern**. It is not the behavior itself, but rather the blueprint used to construct the behavior. The engine's asset loading system identifies the need for this builder based on type information within an NPC's asset file, instantiates it, configures it via the readConfig method, and finally calls build to produce the final HeadMotionObserve instance. This decouples the complex asset loading and validation logic from the runtime component.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderHeadMotionObserve is created dynamically by the NPC asset loading system when it encounters a component of type "HeadMotionObserve" in an NPC's JSON definition. It is never instantiated directly by gameplay logic.
- **Scope:** The object's lifetime is extremely short and confined to the asset deserialization process. It exists only to parse a single JSON object and construct a single HeadMotionObserve instance.
- **Destruction:** The builder is eligible for garbage collection immediately after the build method is called and the resulting HeadMotionObserve object is integrated into the NPC's behavior configuration. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The readConfig method populates member fields like angleRange and pauseTimeRange. This state is a temporary container for the configuration values read from JSON before they are passed to the HeadMotionObserve constructor.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. It must only be used within the context of the main asset loading thread. Concurrent calls to readConfig on the same instance will result in a race condition and corrupted state. The engine guarantees this by processing asset files serially.

## API Surface
The public API is designed for a specific, ordered invocation by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderHeadMotionObserve | O(N) | Parses and validates a JSON object, populating the builder's internal state. N is the number of keys in the JSON. Throws exceptions on validation failure. |
| build(BuilderSupport) | HeadMotionObserve | O(1) | Constructs and returns a new HeadMotionObserve runtime object using the currently configured state. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class via code. Instead, they define the behavior declaratively in an NPC's JSON asset file. The engine's asset system uses the builder internally.

The conceptual flow within the engine is as follows:

```java
// Engine-level code, not for general use.
// 1. Engine finds the correct builder for a JSON component.
BuilderHeadMotionObserve builder = assetRegistry.getBuilderFor("HeadMotionObserve");

// 2. Engine passes the relevant JSON data to the builder.
builder.readConfig(npcComponentJson);

// 3. Engine requests the final runtime object.
BuilderSupport support = createBuilderSupportForNpc(npc);
HeadMotionObserve runtimeComponent = builder.build(support);

// 4. The runtime component is attached to the NPC.
npc.getBehaviorController().addHeadMotion(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new BuilderHeadMotionObserve(). The asset system is responsible for managing builder lifecycles. Direct creation bypasses the engine's registry and will fail to integrate with the NPC.
- **Incorrect Invocation Order:** Calling build before readConfig is a critical error. This will produce a HeadMotionObserve component with default or uninitialized values, leading to unpredictable or inert NPC behavior.
- **Instance Reuse:** A single builder instance is meant for a single build operation. Do not attempt to cache and reuse a builder for multiple different JSON configurations.

## Data Pipeline
The builder is a single, critical step in the data transformation pipeline that turns a static asset file into a live NPC.

> Flow:
> NPC Asset JSON File -> Asset Manager -> JSON Parser -> **BuilderHeadMotionObserve** -> HeadMotionObserve Instance -> NPC Behavior Controller

