---
description: Architectural reference for CraftingRecipe
---

# CraftingRecipe

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class CraftingRecipe implements JsonAssetWithMap<String, DefaultAssetMap<String, CraftingRecipe>> {
```

## Architecture & Concepts

The CraftingRecipe class is a passive data model that represents a single, specific crafting recipe within the game engine. It is not an active service or manager; instead, it serves as the server-side source of truth for the inputs, outputs, and conditions required to craft an item.

Its primary architectural feature is the static **CODEC** field, an instance of AssetBuilderCodec. This codec declaratively defines the entire schema for a crafting recipe asset. It is responsible for:
1.  **Deserialization:** Mapping fields from a JSON asset file to the corresponding fields in a CraftingRecipe Java object.
2.  **Type Conversion:** Ensuring JSON values are correctly converted to Java types like enums, arrays, and primitives.
3.  **Validation:** Enforcing rules and constraints on the asset data, such as ensuring time values are non-negative or that certain configurations are logical. Failures or warnings are reported during the asset loading phase.
4.  **Post-Processing:** Executing logic after the object has been fully decoded, via the `afterDecode` hook which calls `processConfig`.

All CraftingRecipe instances are discovered, loaded, and managed by the Hytale **Asset System**. They are stored in a central, statically-accessible AssetStore, which acts as a global registry. This pattern ensures that recipe data is loaded from disk only once at server startup and is then shared efficiently and immutably across all game systems.

## Lifecycle & Ownership

-   **Creation:** CraftingRecipe instances are created exclusively by the Hytale AssetStore during the server's bootstrap or asset-reloading phase. The static CODEC is invoked for each corresponding JSON asset file found in the game's data directories. **Manual instantiation of this class during runtime is strictly forbidden.**
-   **Scope:** Application-scoped. Once loaded into the AssetStore, a CraftingRecipe instance persists for the entire lifetime of the server process. The data within is considered immutable after the initial load and post-processing.
-   **Destruction:** Instances are dereferenced and become eligible for garbage collection only when the server shuts down or when the AssetRegistry is explicitly cleared, for example during a hot-reload of game assets.

## Internal State & Concurrency

-   **State:** The state of a CraftingRecipe is **effectively immutable**. Its fields are populated once from a JSON file by the AssetBuilderCodec. While the fields are not declared as final, the class provides no public setters and is not designed to be mutated after its initial creation. The private `processConfig` method performs a one-time normalization of state immediately after decoding.
-   **Thread Safety:** The class is **thread-safe for read operations**. Because its internal state does not change after loading, multiple threads (e.g., separate threads for player interactions) can safely access its data without requiring locks or other synchronization primitives. Any attempt to modify an instance after it has been loaded would be an architectural violation and lead to unpredictable behavior.

## API Surface

The public API is designed for read-only access to recipe data and for integration with other core systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the singleton AssetStore that manages all CraftingRecipe instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | Returns the map of all loaded recipes, keyed by their string identifier. |
| toPacket(String id) | com.hypixel.hytale.protocol.CraftingRecipe | O(N) | Converts the server-side data model into a network-optimized packet for client transmission. N is the number of input/output materials. |
| isRestrictedByBenchTierLevel(String, int) | boolean | O(M) | Performs a business logic check to see if the recipe is usable with a given bench and tier level. M is the number of bench requirements. |
| getInput() | MaterialQuantity[] | O(1) | Returns the array of materials required to perform the craft. |
| getOutputs() | MaterialQuantity[] | O(1) | Returns the array of materials produced by the craft. |

## Integration Patterns

### Standard Usage

The correct way to interact with crafting recipes is to retrieve them from the central, statically-accessed asset map. Never create a new instance directly.

```java
// Retrieve the central map of all loaded recipes
DefaultAssetMap<String, CraftingRecipe> allRecipes = CraftingRecipe.getAssetMap();

// Look up a specific recipe by its asset key (e.g., from a JSON file)
CraftingRecipe stoneAxeRecipe = allRecipes.get("mygame:stone_axe_recipe");

if (stoneAxeRecipe != null) {
    // Use the recipe data for game logic, such as validating a player's craft attempt
    if (player.getInventory().hasItems(stoneAxeRecipe.getInput())) {
        // Proceed with crafting logic
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new CraftingRecipe()`. This bypasses the asset loading pipeline, validation, and the central registry. An object created this way will be invisible to the rest of the game engine and will likely be incomplete or invalid.
-   **Runtime State Mutation:** Do not attempt to modify the fields of a CraftingRecipe instance after it has been retrieved from the AssetStore. This violates its immutable nature and can cause severe state inconsistencies across the server.

## Data Pipeline

The CraftingRecipe class is a key component in the data flow from game assets on disk to the client's user interface.

> Flow:
> `recipe.json` (File System) -> AssetStore Loader -> **CraftingRecipe.CODEC** (Deserialization & Validation) -> **CraftingRecipe Instance** (In-Memory) -> AssetStore Registry -> Crafting System (Game Logic) -> `toPacket()` -> Network Layer -> Client Game State

