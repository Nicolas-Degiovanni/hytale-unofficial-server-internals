---
description: Architectural reference for BuilderActionAttack
---

# BuilderActionAttack

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderActionAttack extends BuilderActionBase {
```

## Architecture & Concepts

The BuilderActionAttack class is a configuration-driven factory responsible for constructing an executable ActionAttack instance. It serves as a critical bridge between the declarative NPC behavior definitions stored in JSON assets and the server's runtime behavior system.

Its primary role is to parse a JSON object that defines the parameters of an NPC attack—such as the attack type, timing, and special conditions—and encapsulate this configuration. This builder is not the action itself; rather, it is the blueprint used to create the action.

A key architectural pattern employed is the use of Holder objects (e.g., AssetHolder, FloatHolder, NumberArrayHolder). This abstraction allows configuration values to be either static literals defined directly in the JSON or dynamic values resolved at build time via the BuilderSupport context. This provides significant flexibility, enabling the creation of generic behaviors that can be customized with runtime parameters.

This class operates exclusively during the server's asset loading and behavior compilation phase. The final product, an ActionAttack object, is a lightweight, immutable object designed for high-performance execution within the NPC's behavior tree during the main game loop.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderActionAttack is created by the NPC asset parsing system for each "attack" action defined within an NPC's behavior asset file. Immediately following instantiation, its readConfig method is invoked with the relevant JSON data.
- **Scope:** The lifecycle of a BuilderActionAttack object is ephemeral. It exists only during the parsing and compilation of an NPC's behavior graph. It does not persist into the active game state.
- **Destruction:** The object is eligible for garbage collection as soon as the build method has been called and the resulting ActionAttack instance has been integrated into the runtime behavior tree. It holds no external references and is designed to be discarded after use.

## Internal State & Concurrency
- **State:** The internal state is highly mutable during the configuration phase, as the readConfig method populates its fields from the source JSON. After this phase, the object's state should be considered final and effectively immutable. The state represents the complete static definition of an attack action before it is instantiated for runtime.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access during the asset loading sequence. It must not be shared, modified, or accessed from multiple threads. All parsing and building operations are expected to occur synchronously.

## API Surface

The public API is focused on configuration and instantiation. Accessors are provided to retrieve configured values, often requiring a BuilderSupport context for dynamic resolution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionAttack | O(1) | **Primary factory method.** Constructs and returns the immutable, runtime ActionAttack object. |
| readConfig(JsonElement) | BuilderActionAttack | O(N) | Populates the builder's state from a JSON definition. N is the number of keys in the JSON object. |
| getAttack(BuilderSupport) | String | O(1) | Resolves the attack interaction asset name. May be resolved dynamically via the context. |
| getAttackParameterSlot(BuilderSupport) | int | O(1) | Determines if the attack is defined inline or must be resolved from a dynamic behavior parameter. |

## Integration Patterns

### Standard Usage

The BuilderActionAttack is used exclusively by the server's internal NPC asset loading and behavior compilation systems. A developer defining NPC behavior in JSON implicitly triggers this pipeline.

```java
// This code is a conceptual representation of the server's internal process.
// Developers do not write this code directly.

// 1. The system parses a JSON behavior file.
JsonElement jsonActionDefinition = parseBehaviorFile(".../npc_behavior.json");

// 2. A builder is instantiated and configured.
BuilderActionAttack builder = new BuilderActionAttack();
builder.readConfig(jsonActionDefinition);

// 3. During behavior tree compilation, the builder creates the runtime action.
// The BuilderSupport provides runtime context, like parameter mappings.
ActionAttack runtimeAction = builder.build(builderSupportContext);

// 4. The final action is attached to a node in the NPC's behavior tree.
behaviorTreeNode.setAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Runtime Instantiation:** Never create or use a BuilderActionAttack instance within the main game loop or in response to game events. It is a configuration-time object only. Use the pre-built ActionAttack object for execution.
- **State Mutation After Build:** Do not modify the builder's state or call readConfig after the build method has been invoked. The resulting ActionAttack may be created with inconsistent or incorrect state.
- **Caching Builder Instances:** There is no performance benefit to caching BuilderActionAttack instances. They are lightweight and should be created and discarded during the loading phase as needed.

## Data Pipeline

BuilderActionAttack functions as a step in the configuration data pipeline, transforming declarative text into an executable object.

> Flow:
> NPC Behavior JSON File -> Server JSON Parser -> **BuilderActionAttack.readConfig** -> Populated **BuilderActionAttack** Instance -> **build(BuilderSupport)** -> Executable ActionAttack Object -> NPC Behavior Tree Node

