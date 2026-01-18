---
description: Architectural reference for BuilderActionSetBlockToPlace
---

# BuilderActionSetBlockToPlace

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderActionSetBlockToPlace extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetBlockToPlace class is a factory component within the server-side NPC behavior system. Its primary responsibility is to translate a static data definition, typically from a JSON configuration file, into a live, executable game object.

This class acts as a bridge between the asset configuration layer and the runtime instruction layer for NPCs. It parses a specific JSON structure that defines which block an NPC should place, validates the specified block type against existing game assets, and ultimately constructs an ActionSetBlockToPlace instance. This resulting Action object is then integrated into an NPC's behavior tree or instruction queue to be executed during gameplay.

This pattern decouples the static data format from the runtime implementation, allowing the game's action logic to evolve independently of the configuration files.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level configuration parser, such as a behavior tree factory, whenever it encounters a JSON object corresponding to the "SetBlockToPlace" action. It is not a managed service or singleton.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of parsing a single JSON element and building one corresponding Action.
- **Destruction:** The instance is eligible for garbage collection immediately after the build method is called and its resulting Action is passed to the parent system. It does not persist.

## Internal State & Concurrency
- **State:** The class holds mutable state in the form of the block field, an AssetHolder. This state is populated by the readConfig method and represents the configuration for the single Action instance it is responsible for creating. The state is transient and not shared.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and used in a single-threaded context during the server's asset loading or world initialization phase. Calling readConfig from multiple threads on the same instance will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionSetBlockToPlace instance using the internal state. |
| readConfig(JsonElement) | BuilderActionSetBlockToPlace | O(1) | Parses the provided JSON, validates the "Block" asset, and populates the internal state. This method mutates the object. |
| getBlockType(BuilderSupport) | String | O(1) | Resolves and returns the string identifier for the configured block asset. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for use in development tools. |

## Integration Patterns

### Standard Usage
This class is intended to be used by automated configuration loading systems. The typical flow involves instantiation, configuration via readConfig, and finally, creation of the runtime object via build.

```java
// Hypothetical usage within a larger configuration parser
JsonElement actionJson = getActionConfigFromJson(); // e.g., { "Block": "hytale:stone" }
BuilderActionSetBlockToPlace builder = new BuilderActionSetBlockToPlace();

// Configure the builder from the data
builder.readConfig(actionJson);

// Build the final, executable Action
Action runtimeAction = builder.build(builderSupport);
npcBehaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not call readConfig multiple times on the same instance to produce different Actions. The internal state is not designed for safe re-use and may lead to inconsistent objects. Always create a new builder for each configuration block.
- **Instantiation without Configuration:** Do not instantiate this class and call build without first calling readConfig. This will result in an uninitialized Action that will likely cause a NullPointerException or other runtime error when executed.
- **Concurrent Modification:** Do not access an instance of this builder from multiple threads. The configuration and build process must be atomic and single-threaded.

## Data Pipeline
The BuilderActionSetBlockToPlace serves as a critical transformation step in the NPC behavior definition pipeline. It converts declarative data into an imperative command.

> Flow:
> NPC Behavior JSON -> JSON Parser -> Behavior Tree Factory -> **BuilderActionSetBlockToPlace.readConfig()** -> **BuilderActionSetBlockToPlace.build()** -> ActionSetBlockToPlace Instance -> NPC Instruction Queue

