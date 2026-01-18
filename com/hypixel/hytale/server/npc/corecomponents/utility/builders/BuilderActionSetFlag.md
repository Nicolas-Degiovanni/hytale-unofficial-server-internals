---
description: Architectural reference for BuilderActionSetFlag
---

# BuilderActionSetFlag

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSetFlag extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetFlag is a configuration-time component within the server's NPC Behavior asset pipeline. It is not a runtime game object. Its sole responsibility is to act as a factory, translating a human-readable JSON definition into a compiled, performance-optimized runtime instruction.

This class embodies the "Builder" pattern, specifically for creating an ActionSetFlag instance. It serves as the critical bridge between declarative asset files (JSON) and the server's imperative instruction set that an NPC executes.

The internal state, represented by StringHolder and BooleanHolder, signifies that the configured values are not necessarily static. They can be dynamic placeholders that are resolved during the asset compilation phase using a provided BuilderSupport context. This allows for flexible and reusable NPC behaviors.

A key architectural feature is the pre-computation of the flag's identity. The method getFlagSlot resolves the flag's string name into an integer index. This is a significant optimization, allowing the runtime ActionSetFlag to perform a fast array lookup instead of a slow string comparison or map lookup on every execution.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level asset parser, likely via a factory or reflection, when it encounters a JSON object with a "type" field corresponding to this action. A new instance is created for each "Set Flag" action defined in the asset files.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of parsing and building a single Action.
- **Destruction:** The BuilderActionSetFlag is eligible for garbage collection immediately after the build method is called and the resulting ActionSetFlag object has been integrated into the final, compiled NPC behavior tree. It holds no references and is not persisted.

## Internal State & Concurrency
- **State:** The state is mutable. The readConfig method populates the internal name and value holders. This state is transient and only relevant during the asset build process.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used exclusively within a single-threaded asset loading and compilation pipeline. Accessing an instance from multiple threads will lead to corrupted state and unpredictable build artifacts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs the final, runtime ActionSetFlag object. |
| readConfig(JsonElement) | BuilderActionSetFlag | O(1) | Deserializes the JSON definition into the builder's internal state. |
| getFlagSlot(BuilderSupport) | int | O(log N) | **Critical:** Resolves the flag's string name into a performance-optimized integer slot via the BuilderSupport context. |
| getValue(BuilderSupport) | boolean | O(1) | Resolves the configured boolean value, potentially from a dynamic source. |

## Integration Patterns

### Standard Usage
The builder is used as a temporary object within the asset loading system to produce a permanent, runtime Action.

```java
// Illustrative example of the asset pipeline's logic
JsonElement actionJson = parseActionFromJsonFile();
BuilderSupport support = getAssetCompilationContext();

BuilderActionSetFlag builder = new BuilderActionSetFlag();
builder.readConfig(actionJson);

// The resulting 'runtimeAction' is the optimized object used by the game server
Action runtimeAction = builder.build(support);
npcBehaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not reuse a BuilderActionSetFlag instance to build multiple actions. Each instance is designed to process a single JSON configuration object. Reusing an instance will cause state from the previous configuration to leak into the new one.
- **Premature Build:** Do not call the build method before readConfig has been successfully invoked. This will produce an uninitialized Action that will cause NullPointerExceptions or other undefined behavior at runtime.
- **Runtime Instantiation:** This class has no purpose at runtime. Instantiating it within the main game loop or outside of the asset compilation pipeline is a design error.

## Data Pipeline
The class functions as a transformation step in the data flow from raw asset files to executable server logic.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> `JsonElement` -> `BuilderActionSetFlag.readConfig()` -> **BuilderActionSetFlag Instance** -> `BuilderActionSetFlag.build()` -> `ActionSetFlag` (Runtime Object) -> NPC Behavior Instruction List

