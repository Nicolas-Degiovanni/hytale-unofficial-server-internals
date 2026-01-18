---
description: Architectural reference for BuilderActionSetMarkedTarget
---

# BuilderActionSetMarkedTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSetMarkedTarget extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetMarkedTarget class is a key component of the server-side NPC asset pipeline. It functions as a deserializer and factory, translating a declarative JSON configuration into a concrete, executable runtime object.

Its primary responsibility is to parse a specific "action" block within an NPC's behavior definition file. It validates the provided configuration, such as the target slot name, and prepares the necessary data for the creation of an `ActionSetMarkedTarget` instance. This class acts as the bridge between the static data defined by designers in JSON files and the live, in-game AI system that executes NPC behaviors.

This builder also declares its operational requirements. The call to `requireFeature(Feature.LiveEntity)` signals to the asset validation system that any NPC using this action must possess the `LiveEntity` feature, ensuring system integrity before the game even runs.

### Lifecycle & Ownership
- **Creation:** Instantiated reflectively by the NPC asset loading system. When the system parses an NPC behavior file and encounters an action of this type, it creates a new instance of this builder to handle that specific JSON block.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading process. It exists only to parse its corresponding JSON element and produce a single `Action` instance.
- **Destruction:** Once the `build` method is called and the resulting `ActionSetMarkedTarget` object is integrated into the NPC's runtime behavior tree, this builder instance is no longer referenced and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The internal state is mutable during the configuration phase. The `readConfig` method populates the `targetSlot` field from the source JSON. The `targetSlot` itself is a `StringHolder`, a specialized container designed to hold a value that may be resolved later using an execution context. After `readConfig` completes, the builder's state is effectively immutable.

- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. The asset loading pipeline that utilizes these builders is expected to operate in a single-threaded context for each asset being processed. An instance of this builder should never be shared across threads or reused.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new `ActionSetMarkedTarget` runtime object. |
| readConfig(JsonElement) | BuilderActionSetMarkedTarget | O(N) | Parses the input JSON, populates internal state, and performs validation. N is the number of keys in the JSON object. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its corresponding integer ID using the provided support context. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this Java class directly. Instead, they define its behavior declaratively within an NPC's JSON asset file. The system uses this builder to interpret the configuration.

**Example NPC Behavior JSON:**
```json
{
  "name": "SetTargetToPlayer",
  "type": "SetMarkedTarget",
  "config": {
    "TargetSlot": "LockedTarget"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderActionSetMarkedTarget()`. The asset loading framework is solely responsible for the lifecycle of builder objects.
- **State Re-configuration:** Do not call `readConfig` multiple times on the same instance or manually modify the `targetSlot` field after initial parsing. Builder instances are single-use and should be discarded after building the action.
- **Caching Instances:** Caching and reusing builder instances is not supported and will lead to unpredictable behavior, as their internal state is tied to a specific configuration context.

## Data Pipeline
This builder operates as a specific step in the data transformation pipeline that converts static NPC assets into live game objects.

> Flow:
> NPC Behavior JSON File -> Asset Loading Service -> **BuilderActionSetMarkedTarget.readConfig()** -> **BuilderActionSetMarkedTarget.build()** -> `ActionSetMarkedTarget` Object -> NPC Behavior Tree -> Game World Execution

