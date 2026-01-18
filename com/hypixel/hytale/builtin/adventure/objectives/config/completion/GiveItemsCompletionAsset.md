---
description: Architectural reference for GiveItemsCompletionAsset
---

# GiveItemsCompletionAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.completion
**Type:** Transient Data Asset

## Definition
```java
// Signature
public class GiveItemsCompletionAsset extends ObjectiveCompletionAsset {
```

## Architecture & Concepts
The GiveItemsCompletionAsset is a data-driven configuration object, not an active service. It represents a specific type of reward or consequence for completing an in-game objective within Hytale's Adventure Mode framework.

Its sole architectural purpose is to act as a declarative link between an objective's completion event and a predefined list of items. When an objective associated with this asset is completed, the game's objective system uses the information within this asset to trigger the item-granting logic.

This class is fundamentally defined by its static **CODEC** field. This `BuilderCodec` is the engine that translates a raw data definition (e.g., from a JSON file) into a validated, in-memory Java object. The codec's use of validators, such as `Validators.nonNull` and `ItemDropList.VALIDATOR_CACHE`, is critical for ensuring data integrity at asset load time. This preemptively prevents runtime errors caused by misconfigured objectives, such as those pointing to non-existent item lists.

As it extends ObjectiveCompletionAsset, it is part of a polymorphic system where different completion types (e.g., run a script, grant experience, give items) can be configured interchangeably for any objective.

## Lifecycle & Ownership
- **Creation:** This object is instantiated exclusively by the Hytale Asset System during the server or client bootstrap phase. The static `CODEC` is invoked by the `AssetManager` to deserialize the corresponding configuration from a game data file. It is never created programmatically via `new` in standard game logic.
- **Scope:** The object's lifetime is tied to the asset pool of the currently active game world or zone. It is loaded once and held in memory as an immutable data blob, referenced by the objective it configures.
- **Destruction:** It is eligible for garbage collection when the server shuts down, a world is unloaded, or a hot-reload of game assets is triggered. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal `dropListId` field is populated once during deserialization by the codec. There are no public methods to mutate this state post-creation. This design guarantees that objective rewards are consistent and predictable throughout a session.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, an instance of GiveItemsCompletionAsset can be safely read by any number of threads simultaneously without requiring locks or other synchronization primitives. This is crucial for performance in a multi-threaded game engine where game logic may need to inspect objective data.

## API Surface
The public API is minimal, reflecting its role as a simple data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDropListId() | String | O(1) | Retrieves the unique asset identifier for the ItemDropList to be awarded. |

## Integration Patterns

### Standard Usage
This asset is not used directly in Java code by most developers. Instead, it is defined declaratively in a data file. The game's objective system internally retrieves this asset and uses its data.

A system responsible for granting rewards would interact with it as follows:
```java
// Pseudo-code demonstrating how the engine uses this asset
Objective completedObjective = player.getCompletedObjective();
ObjectiveCompletionAsset completionAsset = completedObjective.getCompletionConfig();

if (completionAsset instanceof GiveItemsCompletionAsset) {
    GiveItemsCompletionAsset itemReward = (GiveItemsCompletionAsset) completionAsset;
    String dropListId = itemReward.getDropListId();
    
    // The ItemSystem uses the ID to look up the asset and grant the items
    ItemSystem itemSystem = context.getService(ItemSystem.class);
    itemSystem.grantDropListToPlayer(player, dropListId);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new GiveItemsCompletionAsset()`. All objective configurations must be defined in asset files to support content creation, modding, and data-driven design. Manually creating instances will bypass the asset system and lead to unmanageable, hardcoded logic.
- **State Modification:** Do not use reflection to modify the `dropListId` field after creation. The engine assumes these assets are immutable, and changing them at runtime can cause inconsistent rewards and desynchronization between server and client.

## Data Pipeline
This component sits at the end of the asset loading pipeline and serves as a data source for runtime game systems.

> Flow:
> AdventureObjective.json -> AssetManager (Deserialization) -> **GiveItemsCompletionAsset** (In-Memory Object) -> ObjectiveSystem (Runtime Lookup) -> ItemSystem (Reward Granting)

