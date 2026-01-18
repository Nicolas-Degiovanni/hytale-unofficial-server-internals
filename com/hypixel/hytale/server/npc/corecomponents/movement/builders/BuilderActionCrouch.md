---
description: Architectural reference for BuilderActionCrouch
---

# BuilderActionCrouch

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionCrouch extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionCrouch class is a component of the server-side NPC Behavior System. It functions as a specialized factory, responsible for deserializing a JSON configuration snippet and constructing a concrete ActionCrouch object.

Within the Hytale engine, NPC behaviors are defined in data-driven asset files, typically JSON. When the server loads an NPC's behavior profile, a generic asset parser encounters various action definitions. This builder acts as the bridge between the raw data definition for a "crouch" action and its executable, in-memory representation. Its sole purpose is to translate a declarative configuration into an imperative game object, ActionCrouch, which can then be processed by the NPC's instruction scheduler.

This class embodies the Builder pattern, isolating the complex construction logic of an Action from its final representation.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionCrouch is created dynamically by a higher-level factory or asset manager during the parsing of an NPC behavior asset file. It is not intended for manual instantiation in game logic.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading process. It exists only to configure and build a single ActionCrouch instance.
- **Destruction:** Once the build method is invoked and the resulting ActionCrouch is returned, the builder instance has served its purpose. It holds no further references and is eligible for garbage collection.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains a mutable internal state via the BooleanHolder named crouching. The readConfig method populates this state from the input JSON, and the build method consumes it. Each instance is configured for a single, specific Action.

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within the asset loading pipeline. Accessing a single instance from multiple threads will lead to race conditions, as calls to readConfig would overwrite the internal crouching state, resulting in unpredictable and corrupt Action objects.

## API Surface
The public API is designed for use by the NPC asset building system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionCrouch instance based on the internal state configured by readConfig. |
| readConfig(JsonElement) | Builder<Action> | O(1) | Parses the provided JSON data to configure the builder's internal state. Specifically, it looks for a "Crouch" boolean property. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this builder, indicating if it is ready for production use. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. It is invoked by the framework during asset deserialization. The conceptual flow is as follows:

```java
// PSEUDOCODE: Illustrates framework-level usage
// An asset loader identifies an action of type "Crouch"
// and retrieves the corresponding builder.

JsonElement crouchActionJson = parseJson("{ \"Crouch\": true }");
Builder<Action> builder = actionFactory.getBuilderFor("Crouch"); // Returns a new BuilderActionCrouch

// The framework configures the builder and then builds the final object.
builder.readConfig(crouchActionJson);
Action crouchAction = builder.build(builderSupport);

// The resulting action is added to an NPC's behavior sequence.
npc.getBehaviorTree().addAction(crouchAction);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not reuse a single BuilderActionCrouch instance to build multiple actions. Its internal state is mutable and not reset after a build. Each new Action requires a new Builder instance.
- **Build Before Configure:** Calling build before readConfig will result in an ActionCrouch constructed with default values (as defined by the BooleanHolder), which may not reflect the intended behavior from the asset file.
- **Direct Instantiation:** Never instantiate this class directly with `new BuilderActionCrouch()`. The asset system is responsible for its creation and lifecycle.

## Data Pipeline
This builder is a key stage in the NPC behavior asset pipeline. It transforms declarative data into an executable object.

> Flow:
> NPC Behavior JSON File -> Asset Loader -> JSON Parser -> **BuilderActionCrouch** -> ActionCrouch Object -> NPC Instruction Queue

