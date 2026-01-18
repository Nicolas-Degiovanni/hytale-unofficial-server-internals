---
description: Architectural reference for BuilderActionLockOnInteractionTarget
---

# BuilderActionLockOnInteractionTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionLockOnInteractionTarget extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionLockOnInteractionTarget is a component of the server-side NPC asset pipeline. It embodies the **Builder Pattern** to translate a declarative JSON configuration into a concrete, executable game logic object.

Its primary function is to parse a specific JSON block that defines an NPC "lock on" action and construct an instance of ActionLockOnInteractionTarget. This class acts as a factory, decoupling the low-level data format (JSON) from the runtime representation of an NPC behavior (the Action object).

This builder is context-aware. The call to `requireInstructionType` ensures that it can only be used within the scope of an *Interaction* instruction block in an NPC's behavior definition. This prevents invalid behavior combinations and enforces a structured design for NPC AI. It is a single-purpose, transient object used exclusively during the asset loading phase.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset parsing system, such as an AssetManager or a behavior tree factory. The specific builder class to use is typically determined by a "type" field within the source JSON data.
- **Scope:** The object's lifetime is extremely short. It is created, configured via `readConfig`, used once to produce an Action via `build`, and is then immediately discarded.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the `build` method returns and the asset loader moves to the next definition. It holds no persistent state beyond the asset loading transaction.

## Internal State & Concurrency
- **State:** The class contains mutable state, primarily the `targetSlot` field. This state is populated by the `readConfig` method and is only considered valid and stable between the completion of `readConfig` and the invocation of `build`.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to operate within a single-threaded asset loading pipeline. The internal state is modified without any synchronization mechanisms, and concurrent access would lead to a corrupted or unpredictable final Action object.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns the final ActionLockOnInteractionTarget instance. |
| readConfig(JsonElement) | BuilderActionLockOnInteractionTarget | O(N) | Parses the input JSON, populates internal state, and validates configuration. N is the number of keys in the JSON object. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured string-based slot name into a runtime integer index using the provided BuilderSupport context. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary of the action's purpose, intended for use in development tools. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the asset loading system. The conceptual usage by the system is as follows.

```java
// Conceptual example of how the asset system uses this builder
JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");
BuilderActionLockOnInteractionTarget builder = new BuilderActionLockOnInteractionTarget();

// 1. Configure the builder from the data source
builder.readConfig(actionJson);

// 2. Build the final, immutable Action object
// The builderSupport object provides context, like resolving string names to IDs
Action lockOnAction = builder.build(builderSupport);

// 3. Add the action to the NPC's behavior tree
npcBehaviorTree.addAction(lockOnAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class in game logic code. NPC behaviors should be defined entirely within JSON asset files.
- **State Re-use:** Do not hold a reference to a builder instance and attempt to re-use it. A new builder must be created for each unique action definition being parsed.
- **Calling Build Before Read:** Invoking `build` before `readConfig` has been successfully called will result in an improperly configured Action object, likely causing runtime exceptions or undefined NPC behavior.

## Data Pipeline
The class functions as a transformation step in the NPC asset loading pipeline, converting structured text data into an executable object.

> Flow:
> NPC Behavior JSON File -> JSON Parsing Service -> Asset Factory -> **BuilderActionLockOnInteractionTarget.readConfig()** -> **BuilderActionLockOnInteractionTarget.build()** -> ActionLockOnInteractionTarget Instance -> NPC Behavior Tree

