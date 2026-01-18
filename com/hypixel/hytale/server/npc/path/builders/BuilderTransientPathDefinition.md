---
description: Architectural reference for BuilderTransientPathDefinition
---

# BuilderTransientPathDefinition

**Package:** com.hypixel.hytale.server.npc.path.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderTransientPathDefinition extends BuilderBase<TransientPathDefinition> {
```

## Architecture & Concepts
The BuilderTransientPathDefinition is a specialized factory component within the server's NPC Asset Definition pipeline. Its sole responsibility is to translate a declarative JSON configuration block into a fully realized, in-memory TransientPathDefinition object. This class acts as a bridge between raw configuration data and the runtime pathfinding system.

It operates as part of a larger, managed framework where a central BuilderManager selects the appropriate builder for a given asset type. This class embodies the **Builder Pattern**, encapsulating the complex logic of parsing, validating, and constructing a path object.

Internally, it employs a compositional design, delegating the management of waypoint lists and scalar values to helper objects like BuilderObjectListHelper and DoubleHolder. This strategy isolates parsing logic and promotes code reuse across different types of asset builders.

## Lifecycle & Ownership
- **Creation:** An instance is created dynamically by the core asset loading system, specifically a BuilderManager, when it needs to process a JSON definition for a transient path. It is never intended for direct manual instantiation.

- **Scope:** The lifecycle of a BuilderTransientPathDefinition instance is extremely short. It exists only for the duration of parsing and building a single path definition from a single JSON block. It is a single-use object.

- **Destruction:** The object holds no persistent references and is eligible for garbage collection immediately after the `build` method has been invoked and the resulting TransientPathDefinition has been returned to the asset system.

## Internal State & Concurrency
- **State:** The internal state of this class is highly **mutable**. Its primary function is to accumulate configuration data—specifically a list of waypoints and a scale factor—during the `readConfig` phase. This state is temporary and directly tied to the source JSON. It does not cache data across different build operations.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading context. Its internal state is modified sequentially, and concurrent access would result in race conditions and corrupt the final constructed object. The parent asset loading framework must enforce serialized access.

## API Surface
The public API is designed for consumption by the automated asset building framework, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | TransientPathDefinition | O(N) | Constructs the final TransientPathDefinition object from the parsed state. N is the number of waypoints. |
| readConfig(JsonElement) | Builder | O(N) | Populates the builder's internal state from the provided JSON data. This is the primary entry point for data. |
| validate(...) | boolean | O(N) | Performs load-time validation of the parsed data against engine rules, delegating to internal helpers. |

## Integration Patterns

### Standard Usage
This class is used implicitly by the server's asset system. A content designer defines a path in a JSON file, and the engine invokes this builder behind the scenes.

A developer would interact with the *result* of the builder, not the builder itself.

```java
// Conceptual example of using the BUILT object
// The BuilderTransientPathDefinition has already run at this point.

// 1. An NPC asset is loaded, which contains the path definition
NPCAsset asset = assetManager.load("my_npc_with_path.json");

// 2. The path is retrieved from the asset
TransientPathDefinition path = asset.getPathDefinition();

// 3. The path is used by the pathfinding or AI system
npcController.followPath(path);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderTransientPathDefinition()`. The asset framework manages the lifecycle and provides necessary context via its factory methods. Direct creation will result in a non-functional object lacking critical context.

- **State Reuse:** Do not attempt to call `readConfig` on the same instance multiple times or otherwise reuse a builder. Each instance is designed to process exactly one configuration block.

- **Concurrent Modification:** Do not access a builder instance from multiple threads. The asset loading pipeline must be single-threaded to guarantee data integrity.

## Data Pipeline
This builder is a key transformation step in the NPC asset data pipeline. It converts structured text data into a usable, in-memory object for the game engine.

> Flow:
> NPC Definition JSON File -> Asset System Parser -> **BuilderTransientPathDefinition** -> TransientPathDefinition (Runtime Object) -> NPC AI & Pathfinding System

