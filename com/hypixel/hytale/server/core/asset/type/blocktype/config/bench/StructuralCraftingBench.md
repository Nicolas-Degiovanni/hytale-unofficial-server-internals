---
description: Architectural reference for StructuralCraftingBench
---

# StructuralCraftingBench

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Transient

## Definition
```java
// Signature
public class StructuralCraftingBench extends Bench {
```

## Architecture & Concepts
The StructuralCraftingBench class is a configuration model, not an active system. It represents the deserialized configuration data for a specific type of in-game crafting station that organizes recipes into a structured, categorized user interface.

This class acts as a data-driven blueprint for the UI and interaction logic of a crafting bench. Its primary role is to define the layout and behavior of the crafting interface, such as which categories are headers, the specific order of recipe categories, and other UI-related flags. It is instantiated by the engine's asset loading and serialization system (Codec) from game data files, typically JSON.

Game systems, particularly the UI rendering layer, query instances of this class to dynamically build the crafting screen presented to the player. It decouples the hard-coded UI logic from the specific configuration of any given crafting station, allowing designers to define complex bench behaviors entirely through data files.

## Lifecycle & Ownership
- **Creation:** An instance is created exclusively by the Hytale Codec framework during the asset loading phase. The static CODEC field defines the deserialization logic. It is **never** instantiated directly with the new keyword. The internal state is finalized by the afterDecode hook which calls the processConfig method.
- **Scope:** The object's lifetime is bound to the parent asset that contains it, typically a BlockType. It persists in memory as long as the corresponding block asset is loaded.
- **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from memory. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The object's state is mutable only during the deserialization process. The processConfig method, called once upon creation, populates internal lookup maps (headerCategoryMap, categoryToIndexMap) for fast querying. After this one-time initialization, the object is treated as **effectively immutable**. Its state is a direct reflection of the source asset file.

- **Thread Safety:** This class is **conditionally thread-safe**. It is safe for concurrent reads after its construction and initialization are complete. The internal collections are not thread-safe for writes, but since all mutation occurs within the single-threaded context of the Codec's afterDecode hook, this is not a concern.

    **WARNING:** Systems accessing this object can and should treat it as a read-only data source. Any external attempt to modify its state after creation will lead to unpredictable behavior and is not supported.

## API Surface
The public API is designed for high-performance, read-only queries, primarily by the UI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isHeaderCategory(String category) | boolean | O(1) | Checks if a given category should be rendered as a main header in the UI. |
| getCategoryIndex(String category) | int | O(1) | Returns the pre-defined sort index for a category. Used to order categories correctly in the UI. |
| shouldAllowBlockGroupCycling() | boolean | O(1) | Returns a flag indicating if the player can cycle through block variations within a recipe slot. |
| shouldAlwaysShowInventoryHints() | boolean | O(1) | Returns a flag that forces inventory requirement hints to always be visible in the UI. |

## Integration Patterns

### Standard Usage
This object is not retrieved directly. It is accessed as a component of a larger asset, such as a BlockType. The primary consumer is the UI system responsible for rendering the crafting interface.

```java
// Example: A UI controller retrieves the bench config from a block
BlockType currentBlock = world.getBlockTypeAt(player.getLookAtPos());
Bench benchConfig = currentBlock.getBenchConfig();

if (benchConfig instanceof StructuralCraftingBench) {
    StructuralCraftingBench structuralBench = (StructuralCraftingBench) benchConfig;
    
    // Use the config to sort and display recipe categories
    List<RecipeCategory> categories = getAvailableCategories();
    categories.sort(Comparator.comparingInt(cat -> structuralBench.getCategoryIndex(cat.getName())));
    
    // Render UI based on the sorted categories...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new StructuralCraftingBench()`. This will create a useless object with null fields, as it bypasses the Codec-driven initialization and the critical `processConfig` step. This will result in `NullPointerException` during gameplay.

- **State Mutation:** Do not attempt to modify the object's state after it has been loaded. The class is designed as a read-only configuration model.

## Data Pipeline
The data for this object originates from a static asset file and flows through the engine to the UI.

> Flow:
> Asset File (.json) -> Engine Asset Loader -> **Hytale Codec Framework** -> Instantiated **StructuralCraftingBench** (with processConfig) -> Attached to BlockType Asset -> UI System Query -> Rendered Crafting Screen

