---
description: Architectural reference for BuilderActionTimeout
---

# BuilderActionTimeout

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTimeout extends BuilderActionWithDelay {
```

## Architecture & Concepts
The BuilderActionTimeout class is a key component of the server-side NPC asset loading pipeline. It functions as a configuration-to-object translator, specifically designed to parse a JSON definition and construct an executable ActionTimeout instance.

This class embodies the Builder pattern, a prevalent design choice within the NPC system. Its primary responsibility is to decouple the complex, declarative nature of JSON asset files from the concrete, imperative logic of the server's behavior system. It inherits delay-handling logic from its parent, BuilderActionWithDelay, and extends it by optionally wrapping another Action.

A critical internal component is the BuilderObjectReferenceHelper. This helper manages the deferred resolution of nested objects, allowing NPC designers to compose complex behaviors by referencing other actions within the JSON configuration. BuilderActionTimeout acts as an intermediary, parsing the configuration for a timed delay and a potential sub-action, and then assembling them into a single, runtime-ready ActionTimeout object.

## Lifecycle & Ownership
- **Creation:** Instances are created dynamically by the central BuilderManager during the server's asset loading phase. When the manager encounters a JSON object with a type field corresponding to ActionTimeout, it instantiates this builder to handle the parsing.
- **Scope:** The lifecycle of a BuilderActionTimeout instance is extremely short and confined to the asset parsing process. It exists only to process a single JSON object.
- **Destruction:** Once the build method is called and the final ActionTimeout object is returned, the builder instance has fulfilled its purpose. It holds no persistent references and is subsequently garbage collected by the JVM. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. Methods like readConfig directly populate instance fields such as delayAfter and the internal action helper. This state is a temporary representation of the configuration read from the JSON source.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be used exclusively within the single-threaded context of the asset loading pipeline. Sharing instances across threads or re-using an instance to parse multiple configurations will result in unpredictable behavior and state corruption.

## API Surface
The public API is designed for use by the NPC asset loading framework, not for general-purpose development.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimeout | O(1) | Constructs the final ActionTimeout object from the parsed state. Throws exceptions on configuration errors. |
| readConfig(JsonElement) | BuilderActionTimeout | O(K) | Populates the builder's internal state from a JSON object. K is the number of keys in the object. |
| validate(...) | boolean | O(N) | Recursively validates the configuration, including any nested actions. N is the complexity of the nested action's validation. |
| getAction(BuilderSupport) | Action | O(1) | Builds and returns the nested Action, if one was defined in the configuration. Returns null otherwise. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class via Java code. Instead, they define its behavior declaratively in an NPC's JSON asset file. The system's BuilderManager handles the instantiation and processing internally.

```json
// Example NPC Behavior JSON
{
  "type": "ActionTimeout",
  "MinDelay": 1.5,
  "MaxDelay": 3.0,
  "DelayAfter": true,
  "Action": {
    "type": "ActionPlayAnimation",
    "animation": "wave"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderActionTimeout()`. The asset pipeline manages the lifecycle. Manually creating one bypasses the framework's validation and context injection.
- **State Reuse:** Do not call readConfig on an instance that has already been configured. Each builder is designed for a single, one-shot build operation.
- **Concurrent Modification:** Do not access a builder instance from multiple threads. The asset loading process is single-threaded by design to ensure deterministic behavior.

## Data Pipeline
This builder acts as a transformation step in the data flow from static assets to live game objects.

> Flow:
> NPC JSON Asset -> GSON Parser -> BuilderManager -> **BuilderActionTimeout.readConfig()** -> **BuilderActionTimeout.build()** -> ActionTimeout Instance -> NPC Behavior Tree Execution

