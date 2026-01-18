---
description: Architectural reference for BuilderActionFlockJoin
---

# BuilderActionFlockJoin

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionFlockJoin extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionFlockJoin is a configuration and construction component within the server-side NPC Behavior system. It follows the classic Builder pattern, designed to translate declarative data from asset files (specifically JSON) into a concrete, executable game object: the ActionFlockJoin.

Its primary role is to act as a factory for a single, specific type of NPC action. It decouples the complex, error-prone process of parsing configuration data from the final, immutable action object used by the NPC's AI at runtime. This class is part of a larger framework of builders (indicated by its parent, BuilderActionBase), where each builder is responsible for a unique behavior or action.

This component is exclusively used during the server's asset loading and initialization phase. It is not involved in the per-tick game loop execution.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset parser or NPC definition factory. This occurs when the system encounters a "FlockJoin" action type within an NPC's behavior configuration file.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of parsing a single action definition from a configuration file.
- **Destruction:** Once the build method is called and the resulting ActionFlockJoin instance is returned, the builder's purpose is complete. It falls out of scope and is eligible for standard garbage collection. It holds no persistent references and is not registered in any global context.

## Internal State & Concurrency
- **State:** The builder maintains a small, mutable internal state, specifically the boolean field **forceJoin**. This state is populated by the readConfig method and is used to parameterize the ActionFlockJoin object during construction. The state is transient and relevant only to a single build operation.

- **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous, single-threaded use during the asset loading pipeline. The readConfig method directly mutates the internal **forceJoin** field. Concurrent access would result in race conditions and unpredictable behavior.

    **WARNING:** Do not share instances of this builder across multiple threads. The asset loading system must ensure that each builder is confined to a single processing thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionFlockJoin | O(1) | Constructs and returns the final, immutable ActionFlockJoin object using the builder's configured state. This is the terminal operation. |
| readConfig(JsonElement) | BuilderActionFlockJoin | O(1) | Parses a JSON fragment to configure the builder's internal state. This method is the primary entry point for populating the builder from asset data. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary of the action's purpose, likely for use in development tools or editors. |
| getLongDescription() | String | O(1) | Provides a detailed explanation of the action's mechanics, conditions, and outcomes. Intended for developer documentation or tooltips. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns an enum indicating the production-readiness of this feature, such as Stable or Experimental. |

## Integration Patterns

### Standard Usage
The builder is used within a larger asset loading process. The typical sequence is instantiation, configuration via `readConfig`, and finalization with `build`.

```java
// Context: Inside an NPC asset loading service
JsonElement actionConfig = ... // Parsed from an NPC behavior file

BuilderActionFlockJoin builder = new BuilderActionFlockJoin();
builder.readConfig(actionConfig);

// The resulting action is then added to the NPC's behavior tree
ActionFlockJoin finalAction = builder.build(builderSupport);
npcBehavior.addAction(finalAction);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not reuse a single builder instance to create multiple actions. Its internal state is mutable and not reset between builds. A new builder must be instantiated for each action defined in the configuration.
- **Configuration After Build:** Calling `readConfig` after `build` has no effect on the already-created action object. The `build` method is a terminal operation.
- **Direct Instantiation without Configuration:** Creating a builder and calling `build` without first calling `readConfig` will result in an action with default parameters (e.g., `forceJoin` will be false), which may not match the intended behavior from the asset files.

## Data Pipeline
This component operates at configuration time, not runtime. Its data flow is concerned with transforming declarative asset data into an in-memory object.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> JsonElement -> **BuilderActionFlockJoin**.readConfig() -> Internal State (forceJoin) -> **BuilderActionFlockJoin**.build() -> ActionFlockJoin Instance -> NPC Behavior Tree

