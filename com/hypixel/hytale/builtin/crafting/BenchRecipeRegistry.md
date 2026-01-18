---
description: Architectural reference for BenchRecipeRegistry
---

# BenchRecipeRegistry

**Package:** com.hypixel.hytale.builtin.crafting
**Type:** Transient

## Definition
```java
// Signature
public class BenchRecipeRegistry {
```

## Architecture & Concepts

The BenchRecipeRegistry is a stateful container responsible for managing all crafting recipes associated with a *single* type of crafting bench. It is not a global registry; instead, an instance of this class is created for each unique crafting bench identifier, such as *hytale:workbench* or *hytale:anvil*.

Its primary architectural role is to serve as a highly optimized, in-memory cache for recipe data. The server may contain thousands of crafting recipes, and querying the master asset list for every crafting operation would be prohibitively slow. This class pre-processes and indexes recipes relevant to its specific bench, enabling near-instantaneous lookups during gameplay.

Key architectural functions include:
- **Scoping:** It isolates recipes by their required bench, defined by the `benchId` provided during construction.
- **Categorization:** It groups recipes into categories (e.g., "Tools", "Weapons") for organized presentation in the user interface.
- **Reverse Indexing:** It builds an `itemToIncomingRecipe` map, which allows for efficient reverse lookups to determine which recipes can produce a given item. This is critical for features like recipe books or "show usage" tooltips.
- **Material Caching:** It aggregates all valid input materials (both by specific item ID and by generic resource type) into fast-lookup sets. This allows the `isValidCraftingMaterial` method to perform O(1) validation checks.

The registry relies on a global, static `CraftingRecipe.getAssetMap()` to resolve recipe identifiers into full `CraftingRecipe` objects. This makes it a consumer of the central asset system.

### Lifecycle & Ownership
- **Creation:** An instance is created programmatically by a higher-level management system, likely a central `CraftingService` or `RecipeManager`. The manager is responsible for creating one `BenchRecipeRegistry` for each unique crafting bench type defined in the game's assets.
- **Scope:** The object's lifetime is tied to the server's asset loading cycle. It persists as long as the crafting bench type it represents is considered valid. If game assets are reloaded at runtime, any existing `BenchRecipeRegistry` instances should be discarded and recreated to reflect the new data.
- **Destruction:** The object has no explicit `destroy` or `close` method. It is a simple Java object that becomes eligible for garbage collection once the owning manager releases its reference.

## Internal State & Concurrency
- **State:** This class is highly **mutable**. Its core purpose is to build and maintain a collection of internal maps and sets that serve as a cache. The state is not final and is explicitly designed to be modified via methods like `addRecipe`, `removeRecipe`, and `recompute`.

- **Thread Safety:** This class is **not thread-safe**. The internal collections are non-concurrent `fastutil` types (`Object2ObjectOpenHashMap`, `ObjectOpenHashSet`). All methods that modify internal state (`addRecipe`, `removeRecipe`, `recompute`) do so without any synchronization.

> **Warning:** Concurrent access from multiple threads will lead to `ConcurrentModificationException` or silent data corruption. All interactions with a `BenchRecipeRegistry` instance **must** be synchronized externally or confined to a single thread, such as the main server game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addRecipe(requirement, recipe) | void | O(1) | Adds a recipe ID to the appropriate category or the uncategorized set. Does not update lookup caches. |
| removeRecipe(id) | void | O(C) | Removes a recipe ID from all categories. C is the number of categories. |
| recompute() | void | O(N*M) | **Critical.** Rebuilds all internal lookup caches. N is the number of recipes, M is the average number of materials. |
| getAllRecipes() | CraftingRecipe[] | O(N) | Collects and returns all registered recipes for this bench. Resolves IDs against the global asset map. |
| getRecipesForCategory(id) | Set<String> | O(1) | Returns all recipe IDs for a given category. |
| getIncomingRecipesForItem(id) | Iterable<String> | O(1) | Returns all recipe IDs that produce the given item. Relies on a pre-computed map. |
| isValidCraftingMaterial(stack) | boolean | O(1) | Checks if an ItemStack can be used as an ingredient in any recipe for this bench. |

## Integration Patterns

### Standard Usage
The standard pattern involves creating, populating, and then computing the registry. This is typically done once at server startup or after an asset reload.

```java
// 1. A manager creates a registry for a specific bench
BenchRecipeRegistry workbenchRegistry = new BenchRecipeRegistry("hytale:workbench");

// 2. The manager iterates through all global recipes and adds relevant ones
for (CraftingRecipe recipe : CraftingRecipe.getAssetMap().getAssets()) {
    BenchRequirement req = findRequirementForBench(recipe, "hytale:workbench");
    if (req != null) {
        workbenchRegistry.addRecipe(req, recipe);
    }
}

// 3. CRITICAL: The caches are built after all recipes are added
workbenchRegistry.recompute();

// 4. During gameplay, the registry is used for fast lookups
if (workbenchRegistry.isValidCraftingMaterial(player.getHeldItem())) {
    // Logic to handle valid material
}
```

### Anti-Patterns (Do NOT do this)
- **State Desynchronization:** Calling `addRecipe` or `removeRecipe` without subsequently calling `recompute`. This will cause the internal lookup caches to become stale. Methods like `isValidCraftingMaterial` and `getIncomingRecipesForItem` will return incorrect results based on the old data.
- **Concurrent Modification:** Accessing a shared `BenchRecipeRegistry` instance from multiple threads without external locking. This will corrupt the internal state of the maps and sets.
- **Premature Querying:** Attempting to call query methods like `isValidCraftingMaterial` before `recompute` has been called for the first time. The internal caches will be empty, and all queries will fail.

## Data Pipeline

The `BenchRecipeRegistry` acts as a data transformation and caching layer. Its primary pipeline is triggered by the `recompute` method.

> **Flow:**
> Global `CraftingRecipe` Assets -> `addRecipe()` calls -> **BenchRecipeRegistry** (Internal Recipe ID Sets) -> `recompute()` -> **BenchRecipeRegistry** (Populated Lookup Caches) -> `isValidCraftingMaterial()` queries

