---
description: Architectural reference for ChangeFarmingStageInteraction
---

# ChangeFarmingStageInteraction

**Package:** com.hypixel.hytale.builtin.adventure.farming.interactions
**Type:** Configuration-Driven Handler

## Definition
```java
// Signature
public class ChangeFarmingStageInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The **ChangeFarmingStageInteraction** is a server-side, data-driven command that modifies the growth state of a farmable block. It is a concrete implementation of the engine's interaction system, designed to be configured within block asset files rather than invoked directly through procedural code.

This class acts as a bridge between a player or system interaction and the underlying world state stored in the Entity Component System (ECS). When an interaction occurs on a block configured with this handler, the engine executes its **interactWithBlock** method. This method is responsible for a complex series of operations:
1.  Locating the target block's chunk and associated data within the **World**.
2.  Retrieving the block's **BlockType** asset definition to access its **FarmingData**.
3.  Querying the world's **ChunkStore** (the ECS) for a **FarmingBlock** component entity associated with the specific block coordinates.
4.  Creating and initializing a new **FarmingBlock** component if one does not already exist. This is a critical "lazy initialization" pattern for performance.
5.  Calculating the new growth stage based on its configured properties (**increaseBy**, **decreaseBy**, **targetStage**).
6.  Updating the **FarmingBlock** component with the new stage and stage set.
7.  Invoking the corresponding **FarmingStageData.apply** method, which enacts the state change in the world (e.g., changing the block model, running effects).

This handler is a prime example of Hytale's data-driven design philosophy. The logic is generic, while the specific behavior—how much to grow, what stage set to use—is defined entirely by data in asset files.

## Lifecycle & Ownership
-   **Creation:** An instance of **ChangeFarmingStageInteraction** is not created at runtime per-interaction. Instead, it is instantiated once by the **BuilderCodec** during server startup when block assets are loaded into memory. The properties of the instance (**targetStage**, **increaseBy**, etc.) are deserialized from the block's configuration file.
-   **Scope:** The object is a stateless singleton relative to its configuration. A single instance exists for each unique configuration in the asset registry. It persists for the entire server session.
-   **Destruction:** The object is garbage collected when the asset registry is unloaded, typically during server shutdown.

## Internal State & Concurrency
-   **State:** The **ChangeFarmingStageInteraction** object itself is effectively immutable after its creation during asset loading. Its configuration fields are not modified during execution. However, its primary method, **interactWithBlock**, is highly stateful as it performs significant modifications to the world state stored within the **ChunkStore**.

-   **Thread Safety:** **CRITICAL WARNING:** This class is not thread-safe. The **interactWithBlock** method performs direct, unsynchronized mutations on the **World** and its underlying ECS stores. It **MUST** only be invoked from the main server thread or the designated world-update thread managed by the engine's scheduler. Calling this from any other thread will lead to world corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its role as an interaction handler. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(log N) | Executes the core logic of changing a farm block's stage. Complexity is driven by ECS lookups. Throws exceptions on critical failures. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Specifies that this interaction's logic is fully server-authoritative. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Its integration is declarative, specified within a block's JSON asset definition. The engine's interaction module reads this configuration and invokes the handler automatically.

The following conceptual example shows how this interaction would be configured on a "wheat" block to advance its growth stage upon receiving a "bonemeal" interaction.

```json
// Conceptual block asset file: wheat.json
{
  "id": "hytale:wheat",
  "interactions": [
    {
      "type": "onUse",
      "item": "hytale:bonemeal",
      "action": {
        "type": "ChangeFarmingStageInteraction",
        "Increase": 1
      }
    },
    {
      "type": "onInteract",
      "action": {
        "type": "ChangeFarmingStageInteraction",
        "Stage": -1, // Set to final stage
        "StageSet": "Harvested"
      },
      "condition": {
        "type": "IsAtFinalFarmingStage"
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ChangeFarmingStageInteraction()`. The object must be constructed by the engine's asset loading and codec system to ensure its properties are correctly configured. Bypassing this will result in a non-functional handler.
-   **External Invocation:** Do not acquire an instance of this class and call **interactWithBlock** manually. This bypasses the engine's cooldown system, predicate checks, and other crucial parts of the interaction pipeline. Interactions must be triggered through the appropriate high-level APIs, such as **InteractionModule**.
-   **State Modification:** Do not attempt to modify the fields of a cached **ChangeFarmingStageInteraction** instance at runtime. These are shared, configured assets and should be treated as immutable.

## Data Pipeline
The flow of data and control for this interaction is server-authoritative and follows a clear path from user input to world-state modification.

> Flow:
> Player Input -> Network Packet -> Server Interaction Module -> **ChangeFarmingStageInteraction** (from Block Asset Config) -> World & ChunkStore (ECS) -> Mutate **FarmingBlock** Component -> **FarmingStageData.apply()** -> World State Change (e.g., Block Model Update)

