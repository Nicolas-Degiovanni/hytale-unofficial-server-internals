---
description: Architectural reference for UseCaptureCrateInteraction
---

# UseCaptureCrateInteraction

**Package:** com.hypixel.hytale.builtin.adventure.farming.interactions
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class UseCaptureCrateInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The UseCaptureCrateInteraction class implements the server-side logic for capturing and releasing NPCs using a specific item, colloquially known as a "capture crate". This class is a concrete implementation within the server's data-driven Interaction System. It is not intended for direct programmatic use but is instead instantiated and configured via asset files, which are deserialized using its static CODEC.

This interaction handler operates in two distinct modes, determined by the metadata of the ItemStack held by the interacting player:

1.  **Capture Mode:** When a player uses an empty capture crate item on a valid NPC, this class is invoked. It validates that the target NPC belongs to one of the configured `acceptedNpcGroupIds`. If valid, it removes the NPC entity from the world and serializes its essential data (such as its role and model) into a `CapturedNPCMetadata` component, which is then attached to the ItemStack.

2.  **Release Mode:** When a player uses a capture crate item that already contains `CapturedNPCMetadata` on a block, this class handles the release. It reads the metadata, spawns a new NPC entity of the appropriate type into the world near the target block, and removes the metadata from the ItemStack, returning it to an empty state.

This class serves as a critical link between several core engine systems:
*   **Interaction System:** Acts as the primary logic handler for a specific interaction type defined in game assets.
*   **Entity Component System (ECS):** Uses a `CommandBuffer` to safely queue entity removal and creation operations, ensuring that world state modifications are synchronized with the main server tick.
*   **Inventory System:** Directly manipulates a player's inventory by adding or removing metadata from an `ItemStack`. This is the primary mechanism for persisting the captured NPC's state.
*   **NPC & TagSet Systems:** Leverages the `NPCPlugin` for spawning and the `TagSetPlugin` for efficiently validating if an NPC is of a capturable type.

A special interaction exists with `CoopBlock` components. If the target block for a release action has a `CoopBlock` component, this class will attempt to place the NPC *inside* the coop as a "resident" rather than spawning it freely in the world.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the server's asset loading framework during startup or hot-reloading. The static `CODEC` field is used to deserialize a configuration file (e.g., a JSON file) into a fully configured Java object. The `afterDecode` hook is critical for post-processing, such as converting asset ID strings into more efficient integer indexes.

- **Scope:** An instance of this class, once loaded, is effectively a stateless singleton for its configured interaction type. It persists for the entire server session. Its internal state is defined at load-time and is considered immutable thereafter.

- **Destruction:** The object is garbage collected when the server shuts down or when its corresponding asset is unloaded.

## Internal State & Concurrency
- **State:** The internal state consists of configuration data loaded from assets (`acceptedNpcGroupIds`, `fullIcon`, etc.). This state is **immutable** after the `afterDecode` process completes. The methods do not modify this internal state; they operate solely on the `InteractionContext` and `CommandBuffer` passed to them.

- **Thread Safety:** This class is **not thread-safe** if its methods are invoked from arbitrary threads. It is designed to be executed exclusively on the main server thread as part of the game loop. All world modifications are funneled through the `CommandBuffer`, which is the designated mechanism for thread-safe, deferred writes to the ECS world state.

**WARNING:** Never call methods on this class from an asynchronous task or a different thread. Doing so will bypass the `CommandBuffer` and lead to severe concurrency issues, including data corruption and server crashes.

## API Surface

The primary API consists of methods overridden from `SimpleBlockInteraction`. These are invoked by the server's Interaction System, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | protected void | O(N) | Handles the logic for interacting with an entity. N is the number of accepted NPC groups. This is the entry point for the **Capture Mode**. Fails the interaction if the target is not a valid, configured NPC. |
| interactWithBlock(...) | protected void | O(1) | Handles the logic for interacting with a block. This is the entry point for the **Release Mode**. Fails the interaction if the held item does not contain NPC metadata. |

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class via Java code. Instead, they define the interaction in an asset file. The engine's Interaction System is responsible for routing player actions to the correct handler instance.

A conceptual configuration might look like this (format is illustrative):
```json
{
  "interactionType": "UseCaptureCrateInteraction",
  "acceptedNpcGroups": [ "hytale:farm_animal_small", "hytale:farm_animal_medium" ],
  "fullIcon": "textures/items/capture_crate_full.png"
}
```

The engine deserializes this into a `UseCaptureCrateInteraction` object and registers it. When a player performs the associated action, the engine invokes the `tick0` or `interactWithBlock` method on the registered object.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UseCaptureCrateInteraction()`. This will create a non-functional object because the `CODEC` and its `afterDecode` hook are bypassed, leaving critical fields like `acceptedNpcGroupIndexes` uninitialized and causing NullPointerExceptions.
- **External Invocation:** Do not call `tick0` or `interactWithBlock` directly. These methods rely on a fully populated `InteractionContext` and a valid `CommandBuffer`, which are only provided by the server's core Interaction System during its processing loop.
- **State Mutation:** Do not attempt to modify the configuration fields of a loaded instance at runtime. These are considered immutable and changing them can lead to unpredictable behavior.

## Data Pipeline

The flow of data and control depends on whether the interaction is a capture or a release.

> **Capture Flow:**
> Player Input (Use Item on NPC) -> Server Interaction System -> **UseCaptureCrateInteraction.tick0** -> Validate NPC via TagSet -> CommandBuffer:Queue(RemoveEntity) -> Modify ItemStack Metadata -> Player Inventory Update

> **Release Flow:**
> Player Input (Use Item on Block) -> Server Interaction System -> **UseCaptureCrateInteraction.interactWithBlock** -> Read ItemStack Metadata -> CommandBuffer:Queue(SpawnEntity) -> Clear ItemStack Metadata -> Player Inventory Update & World State Change

