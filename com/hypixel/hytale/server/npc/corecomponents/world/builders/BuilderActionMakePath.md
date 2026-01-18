---
description: Architectural reference for BuilderActionMakePath
---

# BuilderActionMakePath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionMakePath extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionMakePath class is a factory component within the server-side NPC asset pipeline. It serves as a bridge between static data configuration (JSON) and executable runtime objects. Its sole responsibility is to parse a JSON definition for an NPC action, validate it, and construct an instance of ActionMakePath.

This class embodies the **Builder pattern**, abstracting the complex process of object construction. It does not contain any game logic for pathfinding itself. Instead, it prepares an immutable Action object that the NPC's behavior engine can later execute.

It operates during the server's initial asset loading phase. By using a BuilderObjectReferenceHelper for its `transientPath` member, it participates in a dependency-aware asset system, allowing path definitions to be defined separately and referenced by name. This promotes configuration reuse and modularity.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionMakePath is created by the core asset loading system (e.g., a BuilderManager) whenever it encounters an action of the corresponding type within an NPC behavior JSON file. A new builder is instantiated for each unique action definition in the configuration files.

- **Scope:** The object's lifetime is strictly confined to the asset loading and validation phase. It is a short-lived, transient object.

- **Destruction:** Once the `build` method has been called and the resulting ActionMakePath object is integrated into the runtime behavior tree, the BuilderActionMakePath instance has fulfilled its purpose. It holds no further references and is eligible for garbage collection. It does **not** persist into the main game loop.

## Internal State & Concurrency
- **State:** The internal state is mutable exclusively during the `readConfig` phase. The primary state consists of the `transientPath` helper, which holds a reference to a configured TransientPathDefinition. After configuration is read, the object's state should be treated as immutable.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed to operate within a single-threaded, synchronous asset loading pipeline. Any concurrent invocation of its methods, particularly `readConfig` and `build`, will result in race conditions, corrupted state, and unpredictable server startup failures.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(N) | Constructs the final, immutable ActionMakePath runtime object. N is the complexity of the referenced path definition. |
| readConfig(JsonElement) | BuilderActionMakePath | O(M) | Deserializes the JSON configuration into the builder's internal state. M is the number of properties in the JSON object. |
| validate(...) | boolean | O(N) | Performs load-time validation of the configuration and its referenced path. |
| getPath(BuilderSupport) | TransientPathDefinition | O(N) | Resolves and builds the referenced TransientPathDefinition asset. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java code. Instead, they define its behavior declaratively in a JSON asset file. The server's asset pipeline invokes the builder automatically.

A conceptual JSON definition that would be processed by this builder:
```json
// Example from an NPC behavior asset
{
  "type": "ActionMakePath",
  "Path": {
    "type": "reference",
    "name": "patrol_route_alpha"
  }
}
```

The system would then use the builder like this internally:
```java
// Conceptual engine code. DO NOT replicate.
BuilderActionMakePath builder = new BuilderActionMakePath();
builder.readConfig(jsonElement);
builder.validate(...);
Action runtimeAction = builder.build(builderSupport);
// The 'runtimeAction' is now ready for an NPC to execute.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionMakePath()` in game logic code. The asset system is solely responsible for its creation and lifecycle. Manual instantiation bypasses critical context and validation helpers.

- **State Mutation After Read:** Do not attempt to modify the builder's state after `readConfig` has been called. The object is not designed for reuse or modification.

- **Calling `build` Before `readConfig`:** Invoking `build` on a freshly instantiated builder will result in a NullPointerException or an invalid runtime Action, as its internal references have not been resolved.

## Data Pipeline
The class functions as a specific stage in the transformation of data from disk to a usable in-memory object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderActionMakePath.readConfig()** -> Internal State (Reference to Path) -> **BuilderActionMakePath.build()** -> ActionMakePath Instance -> NPC Behavior Tree

