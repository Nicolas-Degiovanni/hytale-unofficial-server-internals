---
description: Architectural reference for HarvestCropInteraction
---

# HarvestCropInteraction

**Package:** com.hypixel.hytale.builtin.adventure.farming.interactions
**Type:** Configuration Object

## Definition
```java
// Signature
public class HarvestCropInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The HarvestCropInteraction is a stateless, data-driven class that defines a specific server-side game behavior: harvesting a mature crop. It functions as a concrete implementation within the server's broader Interaction System, which uses a strategy pattern to handle player actions on blocks and entities.

This class is not a long-lived service or manager. Instead, it represents a single, deserialized piece of logic. Its primary architectural role is to act as a bridge between a player's input and the core farming mechanics. When a player interacts with a block configured to use this interaction, the server's InteractionModule invokes the `interactWithBlock` method.

A key design characteristic is its role as a *dispatcher*. It does not contain the complex logic for harvesting (e.g., calculating drops, replacing the block, spawning items). Instead, it gathers the necessary context from the world—such as the target block's type and rotation—and delegates the operation to a centralized utility, FarmingUtil. This separation of concerns ensures that the low-level interaction trigger is decoupled from the high-level gameplay logic of the farming system.

Instances of this class are typically defined in external asset files (e.g., JSON) and loaded at runtime via the provided `CODEC`. This allows game designers to define and modify crop harvesting behavior without recompiling the server source code.

## Lifecycle & Ownership

-   **Creation:** HarvestCropInteraction is instantiated by the server's asset loading system during bootstrap or hot-reloading. The static `CODEC` field is used to deserialize configuration data from a block's asset definition into a Java object. It is never created directly using the `new` keyword in gameplay code.
-   **Scope:** The object's lifetime is tied to the asset that defines it, typically a BlockType. A canonical, shared instance is held by the BlockType configuration and persists for the entire server runtime.
-   **Destruction:** The object is eligible for garbage collection only when its owning BlockType is unloaded, which generally occurs on server shutdown.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** This class has no internal member fields to store data. Its behavior is exclusively determined by the arguments passed to its methods during an interaction event. All operations are performed on data owned by the World or the interacting Entity.

-   **Thread Safety:** **Conditionally Thread-Safe.** As a stateless object, a single instance can be safely referenced from multiple threads. However, its core method, `interactWithBlock`, is designed to be executed exclusively on a world's main server thread. It achieves safe world mutation by queuing state changes into a CommandBuffer, which are then applied atomically at the end of the current game tick.

    **WARNING:** Calling `interactWithBlock` from an asynchronous thread is a severe violation of the engine's threading model and will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the server-side harvest logic. Gathers world context and delegates to FarmingUtil. This method is the primary entry point and is invoked by the server's InteractionModule. |
| simulateInteractWithBlock(...) | void | O(1) | A no-op method for client-side prediction. Its empty body indicates that this interaction produces no immediate client-side visual effects before server confirmation. |

## Integration Patterns

### Standard Usage

This class is not intended to be invoked directly by developers. Instead, it is configured within a block's asset definition file. The server's interaction system automatically discovers and executes this logic when a player interacts with the configured block.

The following pseudo-code demonstrates how a block would be configured to use this interaction.

```json
// Example: my_custom_wheat.json (BlockType Asset)
{
  "id": "my_mod:my_custom_wheat_stage_3",
  "parent": "hytale:crop",
  "interactions": [
    {
      // When a player performs a 'USE' interaction (right-click)
      "type": "USE",
      // The server will execute the HarvestCropInteraction logic
      "interaction": {
        "type": "hytale:harvest_crop"
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new HarvestCropInteraction()`. The class is designed to be configured and managed by the asset system. Manual creation bypasses this entire framework.
-   **Manual Invocation:** Avoid calling `interactWithBlock` directly. The server's InteractionModule performs critical prerequisite checks, such as distance, permissions, and cooldowns. Bypassing it can lead to inconsistent or insecure game behavior.
-   **Extending for State:** Do not create subclasses of HarvestCropInteraction that add stateful member variables. This violates the stateless contract of interaction objects and is not compatible with the asset loading system.

## Data Pipeline

The flow of data for a harvesting action begins with player input and terminates with a change in the world state. This class is a central processing step in that pipeline.

> Flow:
> Player Input (USE action) -> Network Packet -> Server InteractionModule -> **HarvestCropInteraction.interactWithBlock** -> FarmingUtil.harvest -> CommandBuffer (Queue Block Change & Item Spawn) -> End-of-Tick World Update

