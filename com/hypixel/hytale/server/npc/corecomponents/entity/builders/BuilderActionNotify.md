---
description: Architectural reference for BuilderActionNotify
---

# BuilderActionNotify

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionNotify extends BuilderActionBase {
```

## Architecture & Concepts

The **BuilderActionNotify** class is a key component of the server-side NPC behavior system. It functions as a configuration-driven factory, responsible for deserializing an NPC action definition from a data source (typically JSON) and constructing a runtime-executable action object.

Its primary architectural role is to decouple game design from engine code. Game designers define NPC behaviors declaratively in data files. The engine's asset pipeline uses this builder class to translate those data definitions into live game logic. **BuilderActionNotify** specifically handles the "Notify" action, which allows one NPC to send a direct message or signal to another.

This class is not the action itself; it is the blueprint used to create the action. The final, executable logic is encapsulated within the **ActionNotify** class, which is produced by the **build** method. This pattern ensures that the configuration and validation logic is cleanly separated from the runtime execution logic.

## Lifecycle & Ownership

-   **Creation:** An instance of **BuilderActionNotify** is created by the NPC asset loading system for each "Notify" action defined within an NPC's behavior configuration file. It is instantiated reflectively or via a factory map based on the action type specified in the data. Immediately after instantiation, its **readConfig** method is called.

-   **Scope:** The object is transient and short-lived. Its entire existence is confined to the asset parsing and loading phase. It holds the deserialized state from the configuration file just long enough to be passed to the **build** method.

-   **Destruction:** Once the **build** method has been called and the resulting **ActionNotify** object has been integrated into the NPC's behavior tree, the **BuilderActionNotify** instance is no longer referenced and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency

-   **State:** The internal state of this class is highly **mutable**. Its fields, such as **message**, **expirationTime**, and **usedTargetSlot**, are populated directly by the **readConfig** method. The object is effectively a temporary data transfer object between the configuration file and the runtime **Action** object.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the single-threaded context of the asset loading pipeline. Concurrent calls to **readConfig** on the same instance will result in a corrupted state and unpredictable behavior.

    **WARNING:** Do not retain references to this builder or access it from any thread other than the one responsible for loading NPC assets.

## API Surface

The public API is designed for use by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderActionNotify | O(N) | Deserializes the JSON configuration for this action, populating the builder's internal state. N is the number of keys in the JSON object. |
| build(BuilderSupport support) | Action | O(1) | Constructs and returns a new, immutable **ActionNotify** instance using the state previously loaded by **readConfig**. |
| getMessage(BuilderSupport support) | String | O(1) | Resolves and returns the configured notification message. |
| getExpirationTime() | double | O(1) | Returns the configured expiration time for the message in seconds. |
| getUsedTargetSlot(BuilderSupport support) | int | O(1) | Resolves the named target slot into its runtime integer ID. Returns Integer.MIN_VALUE if no specific slot is used. |

## Integration Patterns

### Standard Usage

Direct interaction with this class is reserved for the core NPC behavior system. A game designer defines the action in a JSON file, and the engine's parser uses the builder to construct the runtime object.

```java
// Conceptual engine code for parsing an NPC behavior
JsonElement actionJson = parseActionFromJson("...");
String actionType = actionJson.get("type").getAsString(); // e.g., "Notify"

// The engine uses a factory to get the correct builder
BuilderActionBase builder = ActionBuilderFactory.create(actionType);

// The builder is configured from the data
builder.readConfig(actionJson);

// The final, executable action is built and added to the NPC
Action runtimeAction = builder.build(builderSupport);
npc.getBehaviorTree().addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BuilderActionNotify()` in game logic. The behavior system relies on data-driven configuration. All NPC actions should be defined in asset files.

-   **State Reuse:** Do not call **readConfig** multiple times on the same builder instance. Builders are designed to be single-use and should be discarded after the **build** method is called.

-   **Manual Configuration:** Avoid setting fields on the builder manually. This bypasses the validation and data-driven workflow, making the system harder to debug and maintain.

## Data Pipeline

The flow of data from design to runtime execution is linear and unidirectional.

> Flow:
> NPC JSON Asset -> Asset Loader -> **BuilderActionNotify.readConfig()** -> **BuilderActionNotify.build()** -> ActionNotify Instance -> NPC Behavior Tree

