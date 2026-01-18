---
description: Architectural reference for MultipleItemDropContainer
---

# MultipleItemDropContainer

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.container
**Type:** Transient

## Definition
```java
// Signature
public class MultipleItemDropContainer extends ItemDropContainer {
```

## Architecture & Concepts
The MultipleItemDropContainer is a composite node within the server's loot table system. It represents a rule that can generate a variable number of drops by iterating through a collection of child ItemDropContainer instances. This class is fundamental to creating complex drop behaviors, such as a monster dropping between 1 and 3 items from a specific pool, or a chest yielding multiple distinct item stacks.

Its primary architectural role is to act as a multiplier and aggregator. It is configured with a minimum and maximum count (minCount, maxCount) and a list of child containers. When evaluated, it first determines a random iteration count within its configured range. It then executes its child containers for that number of iterations, allowing each child a chance to add its own items to the final drop list.

This component is designed for a data-driven workflow. Its properties are not intended to be set programmatically but are deserialized from server asset files using the provided Hytale Codec. This allows game designers to define and tune loot tables without modifying engine source code.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the asset loading phase. The static CODEC field defines the deserialization logic that maps configuration file keys (e.g., MinCount, MaxCount, Containers) to the object's fields. Manual instantiation in game logic is a design violation.
- **Scope:** The object's lifetime is bound to the asset that defines it. It is loaded into memory by the AssetManager and persists as a read-only configuration object for the duration of the server session or until its parent asset pack is unloaded.
- **Destruction:** The object is eligible for garbage collection when the server shuts down or when the AssetManager unloads the asset pack containing the loot table definition. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state of a MultipleItemDropContainer is **effectively immutable** post-deserialization. The fields minCount, maxCount, and the containers array are populated once by the Codec and are not expected to change during runtime. This makes the object a predictable, reusable piece of configuration data.
- **Thread Safety:** This class is **conditionally thread-safe**. Its internal state is read-only, making it safe for multiple threads to read its configuration simultaneously. However, the core method, populateDrops, mutates an externally provided List. Callers are responsible for ensuring that the List and the DoubleSupplier passed as arguments are used in a thread-safe manner. The object itself contains no locks or synchronization primitives.

## API Surface
The public API is minimal, focusing on the two primary operations of the loot system: calculating a specific drop and querying all possible drops.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateDrops(List, DoubleSupplier, Set) | void | O(N * M) | Core logic method. Calculates a random count (N) between minCount and maxCount, then iterates through its child containers (M) for each count, recursively populating the provided drop list. |
| getAllDrops(List) | List | O(K) | Recursively traverses all child containers to build a comprehensive list of all *possible* items that could drop, ignoring chance and count. K is the total number of descendant nodes. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is invoked by higher-level systems that manage game events and loot resolution. The primary interaction is through asset configuration files.

A conceptual example of how the *engine* uses this class:
```java
// Engine-level code, not typical user code
// Assume 'container' is an ItemDropContainer loaded from an asset
// It could be a MultipleItemDropContainer, a SingleItemDropContainer, etc.

List<ItemDrop> resultingDrops = new ArrayList<>();
DoubleSupplier random = () -> ThreadLocalRandom.current().nextDouble();
Set<String> dropRefs = new HashSet<>(); // Prevents recursive drop loops

// The engine calls populateDrops on the top-level container
container.populateDrops(resultingDrops, random, dropRefs);

// The engine then processes the resultingDrops list
for (ItemDrop drop : resultingDrops) {
    world.spawnItemEntity(drop);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MultipleItemDropContainer()`. The object is not designed to be configured programmatically. All instances must be loaded from asset files via the Codec system to ensure correctness.
- **State Mutation:** Do not modify the minCount, maxCount, or containers fields after the object has been loaded. This violates the data-driven design and can lead to unpredictable and non-deterministic loot behavior across the server.
- **Incorrect Chance Provider:** The provided DoubleSupplier must generate a new random number between 0.0 and 1.0 for each invocation. Using a provider that returns a constant value will break all probability calculations within the drop resolution pipeline.

## Data Pipeline
The flow of data for this class occurs in two distinct phases: asset loading and runtime evaluation.

**1. Asset Loading Pipeline:**
> Asset File (JSON/HOCON) -> Server AssetManager -> `BuilderCodec<MultipleItemDropContainer>` -> **MultipleItemDropContainer Instance** (In-memory representation of the loot rule)

**2. Runtime Drop Resolution Pipeline:**
> Game Event (e.g., Entity Death) -> Loot Resolution Service -> `populateDrops()` -> **MultipleItemDropContainer** -> Recursive Evaluation of Child Containers -> `List<ItemDrop>` -> World System (Spawns items)

