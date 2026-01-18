---
description: Architectural reference for BuilderActionToggleStateEvaluator
---

# BuilderActionToggleStateEvaluator

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionToggleStateEvaluator extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionToggleStateEvaluator is a factory component within the server-side NPC AI framework. It adheres to the Builder design pattern, serving the specific purpose of translating a declarative JSON configuration into a concrete runtime object: the ActionToggleStateEvaluator.

Its primary role is to act as a bridge between the static asset definition of an NPC's behavior and the executable components of its state machine. As a subclass of BuilderActionBase, it is part of a polymorphic system that allows the NPC asset loader to dynamically instantiate and configure various action builders based on the contents of an NPC's definition file.

A critical architectural function is its interaction with the StateHelper via `stateHelper.setRequiresStateEvaluator()`. This call acts as a dependency declaration, signaling to the parent state machine constructor that the resulting state requires a StateEvaluator component to be present and active at runtime. This prevents runtime errors by ensuring the necessary AI infrastructure is in place before the action is ever executed.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system during server startup or when NPC definitions are hot-reloaded. It is discovered and created when the loader encounters the corresponding action type in a JSON behavior file.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single JSON object and building its corresponding runtime action.
- **Destruction:** The object is immediately eligible for garbage collection after the `build` method has been called and its product, the ActionToggleStateEvaluator, has been integrated into the parent state machine. It holds no persistent state or external references.

## Internal State & Concurrency
- **State:** Mutable. The class contains a single boolean field, `enable`, which is populated by the `readConfig` method. This state is transient and serves only as a temporary container for configuration data before the final, immutable action object is constructed.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading pipeline. Concurrent calls to `readConfig` would result in a race condition, corrupting the state of the `enable` field. This design is acceptable as asset processing is a serialized, deterministic operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionToggleStateEvaluator | O(1) | Constructs the final runtime action object using the configured state. |
| readConfig(JsonElement) | BuilderActionToggleStateEvaluator | O(1) | Parses the input JSON, populates internal state, and registers dependencies. Throws if JSON is malformed. |
| isEnable() | boolean | O(1) | Returns the configured state of the `enable` flag. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked transparently by the engine's NPC asset loader. A developer's interaction is limited to defining the action within a JSON file.

```json
// Example NPC Behavior Definition Snippet
{
  "type": "ToggleStateEvaluator",
  "config": {
    "Enabled": false
  }
}
```
The engine's loader would identify the "ToggleStateEvaluator" type, instantiate this builder, pass the `config` object to its `readConfig` method, and then call `build`.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new BuilderActionToggleStateEvaluator()`. The asset pipeline is responsible for the entire lifecycle of builder objects.
- **State Re-use:** Do not attempt to cache and re-use a builder instance. Each instance is single-use and its internal state is specific to one configuration block.
- **Calling Build Before Read:** Invoking `build` before `readConfig` will produce an action with a default `enable` value (false), which will not match the intended configuration from the asset file.

## Data Pipeline
The flow represents the transformation of static configuration data into an executable runtime component for an NPC.

> Flow:
> NPC Definition File (JSON) -> Server Asset Loader -> **BuilderActionToggleStateEvaluator** -> ActionToggleStateEvaluator (Runtime Object) -> NPC State Machine Execution Context

