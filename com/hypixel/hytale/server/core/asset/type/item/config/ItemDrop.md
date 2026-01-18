---
description: Architectural reference for ItemDrop
---

# ItemDrop

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ItemDrop {
```

## Architecture & Concepts
The ItemDrop class is a foundational data model that represents a template for a potential item drop within the game world. It is not an active entity or service, but rather a passive, serializable configuration object. Its primary role is to define the parameters of a drop—what item, how many, and with what specific metadata—as specified in server-side asset files.

The most critical architectural feature of this class is the static **CODEC** field. This `BuilderCodec` instance tightly couples the ItemDrop class to the engine's data serialization and asset loading pipeline. This design ensures that all ItemDrop instances are created, populated, and validated in a consistent and reliable manner directly from configuration sources like loot tables or block definitions. It acts as the blueprint from which concrete `ItemStack` objects are generated at runtime.

**WARNING:** This class is designed to be instantiated exclusively by the engine's asset loading system via its `CODEC`. Manual instantiation is an unsupported pattern that bypasses crucial validation logic.

## Lifecycle & Ownership
- **Creation:** ItemDrop instances are created by the `BuilderCodec` during the server's asset loading phase. The codec reads data from a source (e.g., a JSON or BSON file defining a loot table), instantiates the object, and populates its fields.
- **Scope:** The lifetime of an ItemDrop object is strictly bound to the lifetime of its parent configuration asset. For example, if an ItemDrop is part of a mob's loot table, it will exist in memory as long as that mob's definition is loaded. It is effectively a read-only data fragment of a larger configuration.
- **Destruction:** The object is eligible for garbage collection when its containing asset is unloaded by the AssetManager. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state of an ItemDrop is considered **effectively immutable** post-initialization. While the fields are not declared `final`, the design pattern dictates that they are populated once by the `CODEC` and never modified thereafter. The `getMetadata` method reinforces this by returning a defensive clone of the internal `BsonDocument`, preventing external mutation of the object's state.
- **Thread Safety:** The class is **thread-safe for read operations**. Multiple threads can safely access its properties simultaneously without synchronization. The `getRandomQuantity` method's thread safety is dependent on the caller providing a thread-safe `Random` instance. The ItemDrop object itself introduces no concurrency hazards.

## API Surface
The public API is minimal, focusing on data retrieval and the core logic of quantity calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemId() | String | O(1) | Retrieves the unique identifier of the item to be dropped. |
| getMetadata() | BsonDocument | O(N) | Returns a **defensive clone** of the item's metadata. N is the size of the document. |
| getRandomQuantity(Random) | int | O(1) | Calculates a random quantity between `quantityMin` and `quantityMax` (inclusive). |

## Integration Patterns

### Standard Usage
An ItemDrop is not typically used directly. Instead, a higher-level system, such as a loot service, will hold a reference to it and use it to generate a concrete quantity when a drop event is triggered.

```java
// Example from a hypothetical LootService
public ItemStack generateLoot(ItemDrop dropTemplate, Random random) {
    // Determine the exact number of items to drop
    int quantity = dropTemplate.getRandomQuantity(random);

    // Retrieve the item definition and metadata
    String itemId = dropTemplate.getItemId();
    BsonDocument metadata = dropTemplate.getMetadata();

    // Create the actual item stack to be spawned in the world
    return new ItemStack(itemId, quantity, metadata);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ItemDrop()`. This bypasses the `CODEC`, which is responsible for critical validation, such as ensuring `quantityMin` is not greater than `quantityMax`. All instances must originate from the asset pipeline.
- **State Mutation:** Do not attempt to modify the fields of an ItemDrop after it has been loaded. This can lead to unpredictable and inconsistent behavior across the server, as these objects are treated as shared, read-only configuration.
- **Shared Random Instance:** Do not use a globally shared, non-thread-safe `Random` instance when calling `getRandomQuantity` from multiple threads. This can lead to contention and biased results.

## Data Pipeline
The ItemDrop class is a key component in the configuration-to-gameplay data pipeline. It represents the deserialized, in-memory form of data defined in static asset files.

> Flow:
> LootTable.json Asset File -> Engine Asset Loader -> **ItemDrop.CODEC** (Deserialization & Validation) -> In-Memory **ItemDrop** Instance -> Loot Service -> `getRandomQuantity()` -> `ItemStack` Creation in World

