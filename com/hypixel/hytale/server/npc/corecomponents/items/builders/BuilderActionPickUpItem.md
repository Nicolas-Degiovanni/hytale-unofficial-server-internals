---
description: Architectural reference for BuilderActionPickUpItem
---

# BuilderActionPickUpItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionPickUpItem extends BuilderActionWithDelay {
```

## Architecture & Concepts
The BuilderActionPickUpItem class is a key component of the server-side NPC asset pipeline. It functions as a transient factory, responsible for translating a declarative JSON configuration into a concrete, executable Action object. Its primary role is to bridge the gap between static NPC behavior definitions stored in asset files and the live, in-game Action instances that are executed by the NPC's behavior tree.

This builder parses specific properties related to an NPC picking up an item, such as the effective range, the target inventory slot, and a list of item patterns to match. It introduces a significant behavioral switch via the **Hoover** property.

-   **Standard Mode (Hoover false):** The resulting Action requires an external stimulus to provide a target item, typically from a component like a DroppedItemSensor. The NPC will only attempt to pick up a specific, targeted item entity.
-   **Hoover Mode (Hoover true):** The resulting Action becomes proactive, causing the NPC to automatically acquire any matching items within its range, filtered by the optional Items list.

This class leverages Holder objects (e.g., DoubleHolder, AssetArrayHolder) to encapsulate configuration values. This pattern allows for deferred value resolution and evaluation within a specific ExecutionContext, enabling dynamic NPC behaviors.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionPickUpItem is created by the server's core asset loading system whenever it encounters the corresponding action type within an NPC's JSON behavior definition. It is never instantiated directly by gameplay logic.
-   **Scope:** The lifecycle of a builder instance is extremely short and confined to the asset parsing phase. It exists only to hold state temporarily while the JSON is read and the final Action is constructed.
-   **Destruction:** Once the build method is called and the corresponding ActionPickUpItem instance is returned, the builder has fulfilled its purpose. It holds no further references and becomes eligible for immediate garbage collection.

## Internal State & Concurrency
-   **State:** The internal state of this class is highly **mutable**. The readConfig method directly modifies its internal fields (range, pickupTarget, items, hoover) based on the input JSON. The state is not intended to be read or modified after the initial parsing is complete.

-   **Thread Safety:** This class is **not thread-safe** and must only be used within a single-threaded context. It is designed exclusively for the synchronous asset loading pipeline. Concurrent calls to readConfig or build will result in a corrupted state and unpredictable behavior. No synchronization mechanisms are employed.

## API Surface
The public API is designed for a single, linear workflow: read the configuration, then build the object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderActionPickUpItem | O(N) | Parses the JSON data, populating the builder's internal state. N is the number of properties in the JSON object. Returns itself for chaining. |
| build(BuilderSupport support) | Action | O(1) | Constructs and returns a new ActionPickUpItem instance using the currently configured state. This is the terminal operation for the builder. |
| getItems(BuilderSupport support) | String[] | O(1) | Retrieves the configured item glob patterns. Intended to be called by the ActionPickUpItem during its construction. |
| getHoover() | boolean | O(1) | Retrieves the configured hoover mode state. |
| getRange(BuilderSupport support) | double | O(1) | Retrieves the configured pickup range. |
| getStorageTarget(BuilderSupport support) | ActionPickUpItem.StorageTarget | O(1) | Retrieves the configured inventory storage target. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked internally by the NPC asset loading system. The conceptual flow is as follows.

```java
// This code is conceptual and represents the engine's internal process.

// 1. The asset loader identifies an "ActionPickUpItem" in the JSON file.
// 2. It instantiates the corresponding builder.
BuilderActionPickUpItem builder = new BuilderActionPickUpItem();

// 3. It passes the relevant JSON data to the builder.
JsonElement actionJsonData = getJsonDataForAction();
builder.readConfig(actionJsonData);

// 4. It provides build-time context and constructs the final Action.
BuilderSupport support = createBuilderSupportForNpc();
Action pickUpAction = builder.build(support);

// 5. The resulting action is added to the NPC's behavior tree.
npcBehaviorTree.addAction(pickUpAction);
```

### Anti-Patterns (Do NOT do this)
-   **State Re-use:** Do not re-use a single builder instance to parse and build multiple actions. Each builder is single-use; its state is not reset between calls to build.
-   **Direct Instantiation:** Do not manually instantiate this class in gameplay code. NPC behaviors should be defined entirely within data files. Creating builders at runtime bypasses the asset pipeline and can lead to unmanageable behavior definitions.
-   **Concurrent Modification:** Never access a builder instance from multiple threads. The asset loading process is strictly single-threaded.

## Data Pipeline
The BuilderActionPickUpItem is a critical transformation step in the NPC behavior data pipeline, converting static data into an executable object.

> Flow:
> NPC Behavior JSON Asset -> Server Asset Parser -> **BuilderActionPickUpItem.readConfig()** -> **BuilderActionPickUpItem.build()** -> ActionPickUpItem Instance -> NPC Behavior Tree Execution

