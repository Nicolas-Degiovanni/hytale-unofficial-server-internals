---
description: Architectural reference for BuilderActionAddToTargetMemory
---

# BuilderActionAddToTargetMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionAddToTargetMemory extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionAddToTargetMemory class is a factory component within the server-side NPC AI asset system. Its sole responsibility is to translate a declarative JSON configuration into a concrete, executable runtime object: the ActionAddToTargetMemory.

This class acts as a bridge between the static NPC definition (stored in asset files) and the dynamic, in-game AI engine. During server startup or asset hot-reloading, a central asset manager parses NPC behavior files. When it encounters a JSON object corresponding to this action, it instantiates this builder, configures it via the readConfig method, and then calls build to produce the final Action instance. This Action is then injected into the NPC's behavior tree or state machine.

The call to requireFeature(Feature.LiveEntity) is a critical architectural constraint. It establishes a hard dependency, ensuring that the resulting action is only ever attached to NPCs that possess the LiveEntity feature. This prevents configuration errors and guarantees that the action will have access to the necessary components (like health and a targeting system) at runtime.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system, likely via reflection or a type registry, when it parses a corresponding entry in an NPC's JSON behavior definition. It is never created directly in game logic code.
-   **Scope:** The object's lifetime is extremely short and confined to the asset loading process. It is created, used to build a single Action instance, and then becomes eligible for garbage collection.
-   **Destruction:** Managed automatically by the Java Garbage Collector once the asset loader has finished using it. It holds no persistent state or unmanaged resources.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. It does not define any of its own member variables and does not store any information from the JSON configuration it reads. Its behavior is entirely determined by its type.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. While the broader asset loading system may be multi-threaded, individual instances of this builder can be processed without locks or synchronization primitives.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling and logs. |
| build(BuilderSupport) | Action | O(1) | Factory method. Instantiates and returns a new ActionAddToTargetMemory object. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns Stable, indicating this is a production-ready component. |
| readConfig(JsonElement) | Builder<Action> | O(1) | Validates that the target NPC has the required LiveEntity feature. Throws if the requirement is not met. |

## Integration Patterns

### Standard Usage
A designer or developer would not interact with this Java class directly. Instead, they would specify its use within an NPC's JSON asset file. The engine handles the rest.

**Hypothetical NPC Behavior JSON:**
```json
{
  "ai": {
    "actions": [
      {
        "type": "ActionAddToTargetMemory",
        // No parameters are needed for this specific builder
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderActionAddToTargetMemory()`. The asset pipeline is responsible for its lifecycle. Direct creation bypasses the configuration and validation stages.
-   **Incorrect Feature Set:** Attempting to use this action builder in the JSON configuration for an NPC that lacks the `LiveEntity` feature (e.g., a non-combat, decorative NPC) will result in a server-side error during asset loading, preventing the NPC from spawning correctly.

## Data Pipeline
This component operates at configuration time, not runtime. Its pipeline transforms declarative data into an executable object.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> **BuilderActionAddToTargetMemory** -> ActionAddToTargetMemory Instance -> NPC Behavior Tree

