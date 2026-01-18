---
description: Architectural reference for BlockConditionInteraction
---

# BlockConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class BlockConditionInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The BlockConditionInteraction is a server-side, data-driven component that functions as a conditional gate within the Hytale Interaction System. It is not a standalone service but rather a configurable rule that determines the flow of a larger interaction sequence. Its primary purpose is to test a target block against a predefined set of conditions.

Architecturally, this class embodies the principle of configuration-over-code. It allows game designers to define complex, stateful logic—such as "can this key open this specific type of chest?" or "is this crop ready to be harvested?"—directly in asset files without requiring new Java code.

When an interaction event occurs, this class is invoked to evaluate the state of the target block. Based on the evaluation, it sets the outcome of the interaction to either **Finished** or **Failed**. This outcome is then used by the parent interaction system to decide which subsequent interaction to execute, effectively acting as a conditional branch in a state machine.

The core logic is driven by an array of **BlockMatcher** objects. The interaction is considered successful if *any one* of the matchers in the array returns a positive match against the target block.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the *new* keyword. They are deserialized and instantiated by the Hytale **Codec** system from game asset files (e.g., JSON) at server startup or during a hot-reload. The static CODEC field defines how the object is constructed from its data representation.
- **Scope:** An instance of BlockConditionInteraction is a stateless template. A single instance, loaded from an asset, is shared and reused for every interaction of that type across the entire server. This follows a Flyweight pattern to minimize memory overhead. It persists for the lifetime of the server session.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or when its defining asset is unloaded or reloaded by the AssetManager.

## Internal State & Concurrency
- **State:** This object is **effectively immutable** after its initial creation by the Codec system. Its internal state, primarily the `matchers` array, is configured once during asset loading and is not designed to be modified at runtime.

- **Thread Safety:** The class is inherently **thread-safe**. Due to its immutable nature, its methods can be safely invoked by multiple server worker threads simultaneously without risk of race conditions or data corruption. It performs no internal locking. All state required for evaluation is passed in as method arguments (e.g., World, InteractionContext), ensuring that each invocation is isolated.

## API Surface

The public API is focused on executing the interaction logic. Framework-specific methods for packet serialization are considered internal details and are omitted.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(N) | Server-side entry point. Evaluates the target block against N matchers. Mutates the passed InteractionContext to set the result. |
| simulateInteractWithBlock(...) | void | O(N) | Server-side entry point for simulation or prediction. Behaves identically to interactWithBlock but may be called in different contexts. |

## Integration Patterns

### Standard Usage

This class is not intended to be invoked directly by module or gameplay code. It is automatically executed by the server's core **InteractionModule** as part of a predefined interaction graph. The graph itself is defined in asset files.

A typical scenario involves an item's configuration referencing this interaction:

```json
// Example conceptual asset file (format is illustrative)
{
  "id": "hytale:skeleton_key",
  "onRightClickBlock": {
    "type": "BlockConditionInteraction",
    "Matchers": [
      {
        "Block": { "Tag": "dungeon_chest" }
      }
    ],
    "Next": {
      "type": "OpenContainerInteraction"
    },
    "Failed": {
      "type": "PlaySoundInteraction",
      "sound": "hytale:ui_lock_fail"
    }
  }
}
```
In this example, when a player uses the skeleton key on a block, the system invokes the configured BlockConditionInteraction. If the block has the `dungeon_chest` tag, the interaction state becomes **Finished**, and the `OpenContainerInteraction` is triggered. Otherwise, it becomes **Failed**, and a failure sound is played.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockConditionInteraction()`. The object will be unconfigured and useless. It must be instantiated by the asset loading system via its CODEC.
- **Runtime State Mutation:** Do not attempt to access and modify the internal `matchers` array after initialization. Doing so would violate the Flyweight pattern, is not thread-safe, and will lead to unpredictable behavior for all players.
- **Assuming a Single Matcher:** The logic iterates through an array of matchers and succeeds if *any* of them match. Do not design interactions assuming only the first matcher will be evaluated.

## Data Pipeline

The BlockConditionInteraction acts as a processing and decision node within the server's broader interaction data flow.

> Flow:
> Player Input -> Network Packet -> **InteractionModule** -> `BlockConditionInteraction.interactWithBlock(context)` -> **[Internal Logic: Evaluate Block vs Matchers]** -> `context.getState().state = InteractionState.Finished / Failed` -> **InteractionModule** -> Execute Next or Failed Interaction from Asset Graph

---

