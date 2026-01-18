---
description: Architectural reference for CraftingPlugin
---

# CraftingPlugin

**Package:** com.hypixel.hytale.builtin.crafting
**Type:** Singleton

## Definition
```java
// Signature
public class CraftingPlugin extends JavaPlugin {
```

## Architecture & Concepts
The CraftingPlugin serves as the central bootstrap and management authority for the entire crafting system within the Hytale server. As a subclass of JavaPlugin, it follows a standard engine pattern where a dedicated plugin initializes and orchestrates all related components for a major game feature.

Its primary architectural responsibilities include:

*   **System Registration:** During the server startup phase via its setup method, the plugin registers all necessary Entity Component System (ECS) components, systems, and codecs. This includes the core CraftingManager component, which holds player-specific crafting state, and the PlayerCraftingSystem, which processes crafting logic.
*   **Asset Lifecycle Management:** The plugin is the primary listener for asset loading and unloading events related to CraftingRecipe and Item assets. It intercepts these events to populate and maintain a set of server-wide, in-memory recipe registries. This design decouples the live game from the asset file system, providing a fast, cached lookup mechanism for all crafting-related queries.
*   **Static Data Provider:** It exposes a static API for querying recipe data, such as retrieving all recipes for a specific crafting bench or validating materials. This makes the CraftingPlugin a global service locator for recipe information, preventing other systems from needing to manage their own recipe caches or understand the underlying data structures.
*   **State Synchronization:** The plugin contains the logic for managing a player's known recipes. It provides static methods like learnRecipe and forgetRecipe, which modify a player's persistent data and then trigger the dispatch of network packets (UpdateKnownRecipes) to synchronize the client's state.

In essence, the CraftingPlugin acts as the glue layer between the asset store, the ECS framework, the network protocol, and player data for all crafting functionality.

## Lifecycle & Ownership
- **Creation:** A single instance of CraftingPlugin is created by the server's core plugin loader during the server initialization sequence. The constructor is invoked with a JavaPluginInit context object, which provides access to essential server registries (e.g., ComponentRegistry, EventRegistry). The static singleton instance is set within the constructor.
- **Scope:** The CraftingPlugin is a server-scoped singleton. Its lifecycle is tied directly to the server process; it persists from server startup to shutdown.
- **Destruction:** The plugin is not explicitly destroyed. It is eligible for garbage collection only when the server process terminates and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** The CraftingPlugin is highly stateful, managing several static, mutable collections that represent the global state of all crafting recipes on the server.
    - **registries:** A map of BenchRecipeRegistry objects, which caches and organizes all loaded recipes by the bench they belong to. This is the primary in-memory database for recipe lookups.
    - **itemGeneratedRecipes:** A cache that tracks recipes dynamically generated from Item assets, ensuring they can be correctly unloaded if the parent Item asset is removed.
- **Thread Safety:** **WARNING:** This class is not thread-safe. The static collections (registries, itemGeneratedRecipes) are mutated within event handlers (onRecipeLoad, onItemAssetLoad). The engine's design assumes that these asset-related events are fired sequentially on the main server thread. Public static methods that read from these collections, such as getBenchRecipes, are also assumed to be called from the main thread. Accessing these collections from asynchronous tasks or other threads without external locking will lead to undefined behavior, including ConcurrentModificationException and data corruption. All interactions with this plugin must be synchronized with the main server game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAvailableRecipesForCategory(benchId, categoryId) | Set<String> | O(1) | Retrieves a set of recipe IDs for a specific category within a bench. Returns null if the bench does not exist. |
| isValidCraftingMaterialForBench(benchState, itemStack) | boolean | O(1) | Checks if an item is a valid ingredient for any recipe at the given bench. |
| learnRecipe(ref, recipeId, accessor) | boolean | O(1) | Teaches a player a new recipe, updates their persistent data, and synchronizes the client. |
| forgetRecipe(ref, recipeId, accessor) | boolean | O(1) | Removes a recipe from a player's known recipes and synchronizes the client. |
| sendKnownRecipes(ref, accessor) | void | O(N*M) | Compiles a player's known recipes into a network packet and sends it to the client. Complexity depends on known recipes (N) and lookups (M). |
| get() | CraftingPlugin | O(1) | Returns the singleton instance of the plugin. |

## Integration Patterns

### Standard Usage
Direct interaction with the CraftingPlugin instance is uncommon. Most functionality is triggered indirectly through game mechanics. For example, a player interacting with a crafting bench will trigger systems that use the plugin's static methods to query for valid recipes. The primary direct use case is for other plugins to access the singleton for deep integration.

```java
// Example: A separate plugin needs to check if a recipe is valid for a custom bench.
Set<String> recipes = CraftingPlugin.getAvailableRecipesForCategory("hytale:workbench", "tools");
if (recipes != null && recipes.contains("hytale:wooden_axe")) {
    // Logic here
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new CraftingPlugin()`. The class is a managed singleton created by the server. Always use the static `CraftingPlugin.get()` method to retrieve the instance.
- **External Modification:** Do not attempt to modify the static recipe registries from outside this class. The internal state is managed exclusively by asset event listeners to ensure data integrity.
- **Asynchronous Access:** Do not call any static methods from a separate thread. All API calls must be made from the main server thread to prevent race conditions and data corruption.

## Data Pipeline

The CraftingPlugin sits at the center of two critical data flows: recipe loading and player state synchronization.

**Recipe Loading Pipeline:**
> Flow:
> Asset File (`.json`) -> Server AssetRegistry -> `LoadedAssetsEvent<CraftingRecipe>` -> **CraftingPlugin.onRecipeLoad** -> `BenchRecipeRegistry` (Internal Cache) -> Game Systems (via static getters)

**Player Recipe Learning Pipeline:**
> Flow:
> Player Interaction -> `LearnRecipeInteraction` -> **CraftingPlugin.learnRecipe** -> `PlayerConfigData` (State Change) -> **CraftingPlugin.sendKnownRecipes** -> `UpdateKnownRecipes` Packet -> Network Layer -> Client UI Update

