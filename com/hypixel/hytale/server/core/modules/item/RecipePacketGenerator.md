---
description: Architectural reference for RecipePacketGenerator
---

# RecipePacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.item
**Type:** Utility

## Definition
```java
// Signature
public class RecipePacketGenerator extends AssetPacketGenerator<String, CraftingRecipe, DefaultAssetMap<String, CraftingRecipe>> {
```

## Architecture & Concepts
The RecipePacketGenerator is a specialized, stateless translator component within the server's asset management framework. Its primary function is to convert server-side representations of crafting recipes into network-routable packets for client synchronization.

Architecturally, this class is a concrete implementation of the **Strategy Pattern**, inheriting from the generic AssetPacketGenerator. The core asset system remains agnostic to the specific data format of recipes; it delegates the responsibility of serialization and packet construction to this generator. This decouples the network protocol from the internal asset data model, allowing each to evolve independently.

The generator handles the three fundamental operations required for state synchronization between server and client:
1.  **Initialization (Init):** Sending the complete set of all known recipes to a client, typically upon login.
2.  **Update (AddOrUpdate):** Sending a delta of new or modified recipes, used for live content updates without a client restart.
3.  **Removal (Remove):** Sending a set of keys for recipes that are no longer valid.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level service, most likely an AssetModule or a central AssetService, during the server's bootstrap sequence. It is a dependency of the system responsible for managing CraftingRecipe assets.
-   **Scope:** As a stateless utility, a single instance is sufficient for the entire server lifecycle. It persists for the duration of the server session.
-   **Destruction:** The object is garbage collected when the server shuts down and its owning module is de-referenced. No explicit cleanup methods are required due to its stateless nature.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no instance fields and does not cache data. Each method call operates exclusively on the arguments provided, ensuring predictable, idempotent behavior.
-   **Thread Safety:** The component is inherently **thread-safe**. Its lack of internal state means multiple threads can invoke its methods concurrently without risk of race conditions or data corruption. No synchronization primitives such as locks or atomics are necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Constructs an UpdateRecipes packet of type Init. Serializes the entire provided map of assets. N is the number of assets. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(N) | Constructs an UpdateRecipes packet of type AddOrUpdate. Serializes a delta of new or changed assets. N is the number of loaded assets. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(N) | Constructs an UpdateRecipes packet of type Remove. Contains only the keys of assets to be removed on the client. N is the number of removed keys. |

## Integration Patterns

### Standard Usage
This generator is not intended for direct use by game logic developers. It is invoked internally by the server's asset management system when it detects changes to CraftingRecipe assets.

```java
// Hypothetical usage within an AssetService
// On server startup or after a hot-reload of assets...

// 1. Get all current recipes
Map<String, CraftingRecipe> allRecipes = recipeAssetMap.getAll();

// 2. Get the dedicated generator for this asset type
RecipePacketGenerator generator = assetGenerators.get(CraftingRecipe.class);

// 3. Create the initialization packet
Packet initPacket = generator.generateInitPacket(recipeAssetMap, allRecipes);

// 4. Broadcast the packet to relevant players
networkManager.sendToAll(initPacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new RecipePacketGenerator()`. The asset framework is responsible for creating and providing access to generator instances. Circumventing this can break the asset pipeline.
-   **Incorrect Packet Usage:** Do not call `generateInitPacket` to send a small update. Clients are state machines that expect specific packet types for different synchronization scenarios. Sending an Init packet when an AddOrUpdate is expected can cause the client's recipe book to be completely overwritten, leading to desynchronization or visual glitches.

## Data Pipeline
The RecipePacketGenerator acts as a critical serialization step in the flow of asset data from the server's memory to the game client.

> Flow:
> Server Asset Change (e.g., Mod Load) -> AssetService detects change -> **RecipePacketGenerator** is invoked -> UpdateRecipes Packet is created -> Network System sends Packet -> Client receives Packet -> Client's RecipeManager updates UI

