---
description: Architectural reference for ChoiceItemDropContainer
---

# ChoiceItemDropContainer

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.container
**Type:** Transient

## Definition
```java
// Signature
public class ChoiceItemDropContainer extends ItemDropContainer {
```

## Architecture & Concepts
The ChoiceItemDropContainer is a composite node within the server's item drop system. It models a probabilistic loot table where one or more items are chosen from a weighted collection of other drop containers. This class is a direct implementation of the "roll a die to pick from a list" concept common in game loot systems.

Architecturally, it acts as a branch in a tree-like structure of ItemDropContainer objects. A root container might represent a monster's entire loot table, with a ChoiceItemDropContainer used to handle its "rare drop" pool. The system recursively processes these containers to generate a final, flat list of items to be awarded.

Its primary role is to be defined declaratively in server asset files (e.g., JSON) and deserialized at runtime by the asset pipeline. This allows designers to construct complex, multi-layered loot behaviors without writing code.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during server asset loading. The static CODEC field defines the deserialization logic from a configuration file into a Java object. Manual instantiation is strongly discouraged.
- **Scope:** An instance is loaded once at server startup and typically persists for the entire server session, held by an asset management service. Its state is considered static configuration data.
- **Destruction:** The object is de-referenced and eligible for garbage collection when the server shuts down or performs a full asset reload. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The object is stateful but functionally immutable after its initial creation by the codec. It holds a pre-calculated weighted map of child ItemDropContainer objects and the minimum/maximum number of "rolls" to perform against that map. This internal state is never mutated by its public methods.
- **Thread Safety:** This class is conditionally thread-safe. Because its internal state is read-only after initialization, the same instance can be safely accessed by multiple threads.
    **WARNING:** Thread safety is contingent on the caller. The List and Set arguments passed to the populateDrops method are mutated. Callers must ensure these collections are either thread-confined or properly synchronized if used in a concurrent context.

## API Surface
The public contract is minimal, focused entirely on the calculation of item drops.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateDrops(List, DoubleSupplier, Set) | void | O(N * C) | The primary operational method. Calculates a roll count and recursively populates the provided list with drops from chosen child containers. N is the number of rolls; C is the complexity of the child containers. |
| getAllDrops(List) | List | O(M * C) | Aggregates all possible drops from *all* child containers, ignoring weights and rolls. Used for analysis or UI previews. M is the total number of child containers. |

## Integration Patterns

### Standard Usage
This container is not meant to be used in isolation. A higher-level service, such as a block or entity behavior system, retrieves a pre-configured instance from an asset registry and uses it to generate loot.

```java
// Assume 'lootService' and 'entity' are provided by the game context
// 1. Retrieve the pre-configured container associated with an entity
ChoiceItemDropContainer dropContainer = lootService.getContainerFor(entity);

// 2. Prepare for drop calculation
List<ItemDrop> resultingDrops = new ArrayList<>();
Set<String> dropReferences = new HashSet<>();
DoubleSupplier random = Math::random; // Or a server-side seeded random

// 3. Execute the drop logic
dropContainer.populateDrops(resultingDrops, random, dropReferences);

// 4. Spawn the resulting items in the world
world.spawnItems(resultingDrops);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChoiceItemDropContainer()`. This bypasses the asset system and results in an unconfigured, useless object. All loot tables should be defined in configuration files.
- **State Mutation:** Do not attempt to modify the internal `containers` map or other fields via reflection after the object has been loaded. This can lead to unpredictable and non-deterministic server behavior.
- **Concurrent List Modification:** Do not pass the same `ArrayList` instance to `populateDrops` from multiple threads without external locking. The method is not designed to safely handle concurrent writes to its output parameters.

## Data Pipeline
The ChoiceItemDropContainer is a processing step in the server's loot generation pipeline. It transforms a trigger event into a concrete list of items.

> Flow:
> Game Event (e.g., MobDeathEvent) -> Loot Resolution Service -> Asset Manager lookup -> **ChoiceItemDropContainer::populateDrops** -> Populated List of ItemDrop -> Item Spawning System

