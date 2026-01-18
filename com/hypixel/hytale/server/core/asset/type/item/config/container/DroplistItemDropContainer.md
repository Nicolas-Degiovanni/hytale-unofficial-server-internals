---
description: Architectural reference for DroplistItemDropContainer
---

# DroplistItemDropContainer

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.container
**Type:** Configuration Model

## Definition
```java
// Signature
public class DroplistItemDropContainer extends ItemDropContainer {
```

## Architecture & Concepts
The DroplistItemDropContainer is a specialized implementation of the ItemDropContainer strategy. Its sole purpose is to act as an indirection or a reference to a separate, globally defined ItemDropList asset. This design facilitates a powerful composition-over-inheritance pattern for loot table management.

Instead of embedding a full list of potential item drops directly within an asset, developers can reference a shared, reusable ItemDropList by its unique asset identifier. This dramatically reduces data duplication and centralizes the management of common loot tables. For example, a "Generic Undead Loot" list can be defined once and referenced by dozens of different monster configurations.

At runtime, this class resolves its internal **droplistId** by querying the server's central asset registry, specifically the map provided by ItemDropList.getAssetMap(). It then delegates the actual drop population logic to the container of the resolved ItemDropList. This class is, therefore, a critical link between individual asset configurations and the global loot table system.

### Lifecycle & Ownership
- **Creation:** Instances are never instantiated directly via a constructor in game logic. They are created exclusively by the Hytale Codec system during the server's asset loading phase. The static CODEC field defines the deserialization contract from a data file, such as JSON.
- **Scope:** The lifecycle of a DroplistItemDropContainer is bound to the lifecycle of its parent asset. It persists in memory for the duration of the server session or until a full asset reload is triggered.
- **Destruction:** The object is marked for garbage collection when the server's asset registries are cleared, typically during a server shutdown or a comprehensive asset reload.

## Internal State & Concurrency
- **State:** The internal state is **effectively immutable** post-deserialization. Its primary field, droplistId, is set once by the CODEC and is not modified during runtime. The class itself is a stateless data-holder that forwards requests.
- **Thread Safety:** This class is **conditionally thread-safe**. While its own state is immutable, its methods perform lookups against a global, static asset map (`ItemDropList.getAssetMap()`). Safe concurrent access is predicated on the assumption that the asset map is a read-only structure after the initial server bootstrap.

**WARNING:** Invoking methods on this class from a separate thread while the server's asset system is performing a reload will lead to unpredictable behavior, including NullPointerExceptions or the retrieval of stale data. All interactions should occur on the main game thread or within systems that guarantee asset stability.

## API Surface
The public contract is focused on recursively populating drop lists.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateDrops(drops, chanceProvider, droplistReferences) | void | O(D) | Recursively populates the input list with drops from the referenced ItemDropList. D is the depth of the reference chain. Includes a cycle detection mechanism. |
| getAllDrops(list) | List | O(D) | Recursively populates the input list with all *possible* drops from the referenced ItemDropList, ignoring probabilities. D is the depth of the reference chain. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class in Java code. Instead, they define it declaratively within an asset data file. The system then uses the container polymorphically.

A typical use case involves a higher-level system retrieving a fully-configured asset and requesting its drops, unaware of the specific container implementation.

```java
// A game system retrieves a pre-configured entity asset
EntityConfig skeletonConfig = AssetRegistry.get(EntityConfig.class, "hytale:skeleton");

// The system requests the entity's drops without knowing the container type
// Under the hood, this could be a DroplistItemDropContainer
ItemDropContainer container = skeletonConfig.getDropContainer();
List<ItemDrop> generatedLoot = container.getDrops(); // The container handles the logic

// generatedLoot now contains items resolved from the referenced droplist
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DroplistItemDropContainer()`. This bypasses the asset loading system and results in a non-functional object with a null `droplistId`. All instances must be defined in data files and loaded by the server.
- **Circular References:** Defining droplists that reference each other in a cycle (e.g., A references B, and B references A) is a critical configuration error. While the `populateDrops` method has a safeguard to prevent infinite recursion, this pattern will cause the circular part of the loot table to be silently ignored.

## Data Pipeline
The primary function of this class is to resolve a reference during the data processing stage of loot generation.

> Flow:
> JSON Asset File -> Server Asset Loader (using CODEC) -> **DroplistItemDropContainer** (in-memory object) -> Loot Generation Service -> `populateDrops()` -> `ItemDropList.getAssetMap().getAsset(droplistId)` -> Recursive call on resolved container -> Final `List<ItemDrop>`

