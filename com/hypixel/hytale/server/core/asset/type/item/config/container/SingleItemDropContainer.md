---
description: Architectural reference for SingleItemDropContainer
---

# SingleItemDropContainer

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.container
**Type:** Transient

## Definition
```java
// Signature
public class SingleItemDropContainer extends ItemDropContainer {
```

## Architecture & Concepts
The SingleItemDropContainer is a specialized data model representing the simplest form of a loot drop rule: a container that holds exactly one potential item. It serves as a concrete implementation within a polymorphic system built around the abstract ItemDropContainer base class. This design allows the server's loot generation services to process various drop configurations—from single items to complex, weighted lists—through a unified interface.

Its primary role is to act as a data-transfer object (DTO) that is deserialized from server asset files (e.g., JSON definitions for block drops or entity loot tables). The static CODEC field is the critical integration point with the engine's asset loading pipeline, defining how raw configuration data is mapped into a living SingleItemDropContainer object.

This class embodies the "leaf" node in a potential composite structure for loot tables. Systems that calculate drops do not need to know the specific type of container they are processing; they simply invoke the contract defined by ItemDropContainer, and this class provides the simplest possible implementation.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale asset loading system during server startup or asset hot-reloading. The static CODEC field is used to deserialize a configuration block into a new SingleItemDropContainer object. Direct programmatic instantiation is rare and typically reserved for unit testing.
-   **Scope:** The object's lifetime is bound to the asset that defines it, such as a monster's configuration or a specific loot table. It is a short-lived, passive data object, not a managed service.
-   **Destruction:** The object is marked for standard Java garbage collection once the owning asset is unloaded or the loot calculation that required it has completed. There is no manual destruction or cleanup process.

## Internal State & Concurrency
-   **State:** This class holds a mutable reference to an ItemDrop object. However, it is designed to be effectively immutable after its initial construction by the codec system. The internal state is established once during deserialization and is not expected to change during its lifetime.
-   **Thread Safety:** **This class is not thread-safe.** It is a simple data holder intended for use within a single-threaded context, such as the primary server game loop. Concurrent modification or access from multiple threads without external locking mechanisms will result in undefined behavior and is strictly prohibited.

## API Surface
The public API is minimal, focusing on fulfilling the contract of its parent class, ItemDropContainer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDrop() | ItemDrop | O(1) | Returns the single, non-null ItemDrop instance held by this container. |
| populateDrops(List, DoubleSupplier, Set) | void | O(1) | Implements the core loot generation logic. Adds its single ItemDrop to the provided output list. |
| getAllDrops(List) | List<ItemDrop> | O(1) | Provides a comprehensive list of all possible drops from this container, which is always a list containing one element. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. Instead, they interact with the abstract ItemDropContainer obtained from a higher-level configuration. The loot generation system processes the container polymorphically.

```java
// Pseudo-code demonstrating engine-level usage
// An entity's configuration holds a generic ItemDropContainer
ItemDropContainer container = entityConfig.getLootContainer();

List<ItemDrop> resolvedDrops = new ArrayList<>();
Set<String> dropReferences = new HashSet<>();

// The engine calls populateDrops, which dispatches to the correct
// implementation (e.g., SingleItemDropContainer)
container.populateDrops(resolvedDrops, Math::random, dropReferences);

// The resolvedDrops list now contains the item from this container
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SingleItemDropContainer()`. Game logic should be data-driven. To define a new single item drop, modify the corresponding server asset file. Instantiating this class directly creates "invisible" logic that is disconnected from the asset pipeline.
-   **Post-Construction Mutation:** Do not retrieve the container and attempt to modify its internal ItemDrop. Configurations are expected to be static after being loaded. Modifying them at runtime can lead to inconsistent loot behavior across the server.

## Data Pipeline
This class is a key component in the server's loot generation data pipeline. It represents a deserialized rule that is ready for processing.

> Flow:
> Loot Table Asset File (JSON) -> Engine Codec Deserializer -> **SingleItemDropContainer Instance** -> Loot Generation Service -> `populateDrops()` call -> Finalized List of ItemDrop objects -> World Drop or Player Inventory

