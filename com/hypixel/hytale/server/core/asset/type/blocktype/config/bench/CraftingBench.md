---
description: Architectural reference for CraftingBench
---

# CraftingBench

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Configuration Model

## Definition
```java
// Signature
public class CraftingBench extends Bench {
```

## Architecture & Concepts
The CraftingBench class is a server-side configuration model that defines the structure and properties of a specific type of crafting station. It is not a system with logic, but rather a pure data-container that represents the deserialized state of a block's asset definition file, likely a JSON file.

This class extends the base Bench class, inheriting common properties of interactive blocks, and specializes it by adding a hierarchical structure of crafting categories. Its primary role is to provide the game's crafting systems with a structured, in-memory representation of what recipes and UI layouts a specific crafting bench block should present to the player.

The static final field CODEC is the most critical component of this class. It leverages the Hytale Codec system to define a declarative schema for serialization and deserialization. The engine's Asset Loading service uses this CODEC to automatically parse the raw asset data and instantiate fully-formed CraftingBench objects. This pattern decouples the data definition from the asset loading logic, allowing for robust and maintainable asset pipelines.

## Lifecycle & Ownership
- **Creation:** CraftingBench instances are created exclusively by the Hytale Codec system during the server's asset loading phase. The static CODEC field is invoked by an AssetManager which reads a corresponding block configuration file from disk. Manual instantiation is an anti-pattern and will result in an uninitialized, non-functional object.

- **Scope:** An instance of CraftingBench has a scope tied to its corresponding block asset. It is loaded once at server startup and cached in a central asset registry. It persists for the entire server session. The same instance is shared and referenced by all game systems whenever they need to query the properties of that specific crafting bench type.

- **Destruction:** The object is eligible for garbage collection only when the server is shutting down and the asset registries are cleared.

## Internal State & Concurrency
- **State:** The state of a CraftingBench object is **Effectively Immutable**. After being deserialized and constructed by the Codec system, its internal fields, such as the categories array, are not intended to be modified at runtime. It serves as a read-only representation of the on-disk asset.

- **Thread Safety:** The class is **Thread-Safe for Read Operations**. As its state is not mutated after initialization, multiple threads from different game systems (e.g., UI, player interaction, recipe validation) can safely access the same CraftingBench instance without synchronization.

    **Warning:** Writing to the object's fields or modifying the contents of the array returned by getCategories after initialization is a severe anti-pattern that will lead to unpredictable behavior and data corruption across the server.

## API Surface
The public API is minimal, designed for data retrieval only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCategories() | BenchCategory[] | O(1) | Returns the array of crafting categories defined for this bench. |

## Integration Patterns

### Standard Usage
A game system, such as a BlockInteractionManager, should retrieve the CraftingBench configuration from a central asset registry using the block's type identifier. The data is then used to populate UI models or validate crafting actions.

```java
// Conceptual example of retrieving the configuration for a given block
Block block = world.getBlockAt(position);
BlockTypeConfig config = assetRegistry.get(block.getType());

// Safely cast after checking the config type
if (config.bench instanceof CraftingBench) {
    CraftingBench benchConfig = (CraftingBench) config.bench;
    BenchCategory[] categories = benchConfig.getCategories();
    
    // Use categories to build the player's crafting UI
    player.getUI().showCraftingScreen(categories);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new CraftingBench()`. The object will be missing its essential category data, which is only populated through the CODEC deserialization process. This will result in NullPointerExceptions and incorrect game behavior.

- **Runtime State Mutation:** Do not modify the array returned by getCategories. This array is a direct reference to the cached asset state. Modifying it will affect every player and system interacting with this type of crafting bench, leading to server-wide inconsistencies.

## Data Pipeline
The CraftingBench class is a key component in the data flow from asset files to the player's game client.

> Flow:
> Block Asset File (JSON) -> Server Asset Loader -> **CraftingBench.CODEC** -> **CraftingBench Instance** (Cached in Registry) -> Block Interaction Logic -> UI Data Packet -> Client UI Rendering

