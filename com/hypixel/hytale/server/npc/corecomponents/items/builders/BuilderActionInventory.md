---
description: Architectural reference for BuilderActionInventory
---

# BuilderActionInventory

**Package:** com.hypixel.hytale.server.npc.corecomponents.items.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionInventory extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionInventory class is a key component of the server-side NPC (Non-Player Character) asset system. It functions as a **Configuration-to-Object Bridge**, translating declarative NPC behavior definitions from JSON into executable runtime objects.

Within the NPC behavior framework, actions are defined in data files. This class is responsible for parsing the specific JSON block that defines an inventory manipulation action (such as adding, removing, or equipping an item). It encapsulates the logic for reading properties, applying default values, and performing initial validation.

Its primary architectural purpose is to decouple the data representation (JSON) from the imperative logic of the final `ActionInventory` object. The builder consumes the raw data and produces a validated, ready-to-use `Action` instance that is then integrated into an NPC's behavior tree or state machine.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's NPC asset loading pipeline when it encounters an `ActionInventory` type in an NPC's JSON configuration. **WARNING:** Manual instantiation by developers is an anti-pattern and will lead to unconfigured, non-functional action objects.
- **Scope:** Extremely short-lived. An instance of BuilderActionInventory exists only for the duration of parsing a single JSON action block.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and the resulting `ActionInventory` object is registered with the parent NPC behavior component. It does not persist into the active game state.

## Internal State & Concurrency
- **State:** The internal state is highly mutable during the configuration phase. Fields like `operation`, `item`, and `count` are held in specialized `Holder` objects. These holders are populated by the `readConfig` method. After configuration, the state is effectively immutable and is only read by the `build` method and the subsequent `ActionInventory` object. The use of `Holder` types allows values to be either static or resolved dynamically at runtime via an `ExecutionContext`, enabling complex, expression-based NPC behaviors.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single, synchronized thread during the NPC asset loading process. Concurrent calls to `readConfig` or other methods will result in a corrupted state and unpredictable behavior.

## API Surface
The public API is intended for use by the NPC asset system, not for direct developer interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new `ActionInventory` instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<Action> | O(1) | Parses the provided JSON, populating the builder's internal state. Throws exceptions on malformed data. |
| validate(...) | boolean | O(1) | Performs load-time semantic validation, such as checking if a specified inventory slot is valid for the given operation. |
| getOperation(BuilderSupport) | ActionInventory.Operation | O(1) | Runtime accessor for the configured operation. Used by the created `ActionInventory` object. |
| getItem(BuilderSupport) | String | O(1) | Runtime accessor for the configured item asset name. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java. Instead, they define the behavior declaratively in an NPC's JSON asset file. The system then uses this builder internally.

A typical JSON definition that would be processed by this builder:
```json
{
  "type": "ActionInventory",
  "Operation": "Add",
  "Item": "hytale:stone_sword",
  "Count": 1,
  "UseTarget": false
}
```

The asset loader would then perform an operation conceptually similar to this:
```java
// Conceptual example of internal system usage
BuilderActionInventory builder = new BuilderActionInventory();
builder.readConfig(jsonElementFromNpcFile);
// ... validation steps ...
Action inventoryAction = builder.build(builderSupport);
npcBehaviorTree.addAction(inventoryAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionInventory()` in game logic. The NPC asset system is the sole owner and creator of builder instances.
- **Configuration Omission:** Do not call `build` before `readConfig`. This will produce an `ActionInventory` object with default values, which will almost certainly not match the intended behavior and can cause subtle bugs.
- **Instance Re-use:** Do not attempt to re-use a single builder instance to parse multiple JSON blocks. Each action definition requires a new, clean builder instance to ensure state is not carried over.

## Data Pipeline
The flow of data from configuration to execution is strictly defined and moves from a declarative format to an executable object.

> Flow:
> NPC Behavior JSON File -> Server JSON Parser -> NPC Asset Loader -> **BuilderActionInventory.readConfig()** -> **BuilderActionInventory.build()** -> `ActionInventory` Instance -> NPC Behavior Tree -> Game Loop Execution

