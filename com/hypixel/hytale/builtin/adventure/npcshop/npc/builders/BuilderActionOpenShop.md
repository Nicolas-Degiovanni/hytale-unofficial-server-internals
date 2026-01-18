---
description: Architectural reference for BuilderActionOpenShop
---

# BuilderActionOpenShop

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionOpenShop extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionOpenShop class is a factory component within the server-side NPC asset pipeline. It is not a runtime object that performs an action, but rather the configuration-time builder responsible for constructing the runtime action. Its primary role is to deserialize a specific JSON configuration block from an NPC asset file and produce a concrete ActionOpenShop instance.

This class acts as a translation layer between static data (JSON assets) and executable game logic (the Action object). It validates the asset-defined parameters, such as the target shop ID, ensuring that the resulting runtime action is correctly configured and linked to existing game assets before it is ever attached to an NPC. This pattern decouples the game's content definition from its implementation, allowing designers to configure complex NPC behaviors without modifying engine code.

## Lifecycle & Ownership
The lifecycle of a BuilderActionOpenShop instance is extremely short and confined to the asset loading phase.

- **Creation:** Instantiated by the NPC asset loading system when it encounters an action of type "OpenShop" within an NPC's JSON definition. It is never created directly by game logic.
- **Scope:** Transient. An instance exists only for the duration required to parse its corresponding JSON block and build a single ActionOpenShop object.
- **Destruction:** The instance is eligible for garbage collection immediately after the call to its build method completes. It holds no persistent state and is not registered with any service locator.

## Internal State & Concurrency
- **State:** The class is stateful but its state is transient. The primary state is the shopId field, an AssetHolder which is populated during the call to readConfig. This state is essential for the subsequent build operation but is discarded with the instance itself.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during asset deserialization. The sequence of operations—creation, `readConfig`, then `build`—is strictly linear and modifies the internal state of the object. Concurrent access would lead to unpredictable behavior and race conditions.

## API Surface
The public API is minimal, designed for use exclusively by the NPC asset pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionOpenShop instance. Throws if called before readConfig. |
| readConfig(JsonElement) | BuilderActionOpenShop | O(N) | Deserializes the JSON data, populates internal state, and performs validation. N is the number of properties. |
| getShopId(BuilderSupport) | String | O(1) | Resolves and returns the configured shop asset ID from the internal AssetHolder. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's internal asset processing systems. A developer would never instantiate or call methods on this class directly. The following example illustrates the conceptual engine-level workflow.

```java
// Conceptual example of engine-level usage
// This code does not exist in this form but shows the pattern.

JsonElement actionJson = parseNpcAssetFile("...").getAction("onInteract");
BuilderActionOpenShop builder = new BuilderActionOpenShop();

// 1. Configure the builder from the asset data
builder.readConfig(actionJson);

// 2. Build the final, executable runtime action
Action runtimeAction = builder.build(assetBuilderSupport);

// 3. Attach the action to the NPC
npc.getInteractionHandler().addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderActionOpenShop()` in game logic. This class is not a runtime component. Its purpose is strictly limited to the asset loading pipeline.
- **State Manipulation:** Do not attempt to modify the internal shopId field after it has been set by readConfig. The object's state is considered sealed after configuration.
- **Calling Build First:** Calling the build method before readConfig has been successfully executed will result in an action with an uninitialized state, leading to runtime exceptions or assertion failures when the action is triggered.

## Data Pipeline
The BuilderActionOpenShop serves as a critical validation and transformation step in the NPC data pipeline.

> Flow:
> NPC JSON Asset File -> Server Asset Parser -> **BuilderActionOpenShop** (`readConfig` validates shop exists) -> `build()` -> ActionOpenShop (Runtime Object) -> NPC Interaction Component

---

