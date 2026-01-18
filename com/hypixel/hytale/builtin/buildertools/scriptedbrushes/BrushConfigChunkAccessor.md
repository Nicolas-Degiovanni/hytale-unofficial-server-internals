---
description: Architectural reference for BrushConfigChunkAccessor
---

# BrushConfigChunkAccessor

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes
**Type:** Transient

## Definition
```java
// Signature
public class BrushConfigChunkAccessor extends LocalCachedChunkAccessor {
```

## Architecture & Concepts
The BrushConfigChunkAccessor is a specialized implementation of the Decorator pattern, designed to provide a transactional, multi-layered view of world data for a single, isolated builder tool operation. It acts as an intermediary between a brush script's logic and the canonical world state.

Its primary function is to intercept block read requests and resolve them against a specific history of changes encapsulated within a BrushConfigEditStore. This architecture enables two critical features for scripted brushes:

1.  **Read Consistency:** The brush script can read from a consistent snapshot of the world as it existed at the *moment the operation began*. This prevents race conditions where other world changes might affect the outcome of the script mid-execution.
2.  **Transactional Awareness:** The script can immediately read its own writes *within the same operation*. Changes made by the brush are reflected in subsequent reads from the accessor, even though they have not yet been committed to the global world state.

By inheriting from LocalCachedChunkAccessor, it also benefits from an underlying chunk cache, ensuring that repeated reads within a localized area are highly performant. This component is fundamental to providing a stable and predictable environment for complex, multi-block construction scripts.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively via the static factory methods, either **atWorldCoords** or **atChunkCoords**. This is performed by the builder tool system at the start of a brush execution cycle. A new instance is created for every distinct brush operation.
-   **Scope:** The object's lifetime is strictly bound to a single brush operation. It is instantiated, used for all world queries by the script, and then discarded once the operation completes.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the brush operation concludes and all references to it are released. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is stateful. It holds a final reference to a BrushConfigEditStore, which contains the mutable before-and-after state of the world for the current operation. While the accessor's own configuration is immutable post-construction, the data it presents is dynamic and reflects the ongoing changes within the edit store.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, used, and discarded within the context of a single thread, typically the main server thread responsible for game logic. The underlying BrushConfigEditStore is not designed for concurrent modification or access.

    **Warning:** Sharing an instance of BrushConfigChunkAccessor across multiple threads will lead to race conditions, data corruption, and unpredictable behavior. All interactions must be externally synchronized by the calling system.

## API Surface
The public API is primarily composed of static factories for creation and overridden methods for data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| atWorldCoords(...) | BrushConfigChunkAccessor | O(N) | **Static Factory.** Creates an accessor for a region defined by world coordinates and a block radius. Complexity is relative to the number of chunks (N) in the radius. |
| atChunkCoords(...) | BrushConfigChunkAccessor | O(N) | **Static Factory.** Creates an accessor for a region defined by chunk coordinates and a chunk radius. |
| getBlock(x, y, z) | int | O(1) | Returns the block ID at the given world coordinates, following the transactional lookup priority. |
| getBlockIgnoringHistory(x, y, z) | int | O(1) | Bypasses the *after* state to read the world as it was when the operation began. |
| getFluidId(x, y, z) | int | O(1) | Returns the fluid ID, which is resolved directly from the associated edit operation. |

## Integration Patterns

### Standard Usage
The intended use is to wrap a delegate ChunkAccessor at the beginning of a tool operation. This new accessor is then used for all subsequent world reads for the duration of that operation.

```java
// 1. Get the primary world accessor and create an edit store for the operation
ChunkAccessor<WorldChunk> worldAccessor = universe.getWorldAccessor();
BrushConfigEditStore editStore = new BrushConfigEditStore();

// 2. Create the transactional accessor for a specific area of effect
BrushConfigChunkAccessor transactionalAccessor = BrushConfigChunkAccessor.atWorldCoords(
    editStore,
    worldAccessor,
    player.getPosition().x,
    player.getPosition().z,
    brush.getRadius()
);

// 3. Pass this accessor to the script logic for all block queries
// The script can now read from a consistent, transaction-aware view of the world.
brushScript.execute(transactionalAccessor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to use `new BrushConfigChunkAccessor()`. The constructor is protected and its direct use will bypass necessary setup logic. Always use the provided static factory methods.
-   **Lifecycle Mismanagement:** Do not cache or reuse a BrushConfigChunkAccessor instance across multiple, distinct brush operations. Each instance is tied to a specific BrushConfigEditStore and will provide a stale and incorrect view of the world if reused.
-   **Writing to Delegate:** Do not attempt to write to the original, delegate ChunkAccessor while a transactional operation is in progress. All modifications should be routed through the BrushConfigEditStore to maintain data consistency.

## Data Pipeline
The core value of this class is its prioritized data lookup pipeline for block reads. When `getBlock` is called, it attempts to resolve the block ID in a specific, hierarchical order.

> Flow:
> Brush Script Call `getBlock(pos)`
> -> **BrushConfigChunkAccessor**
> -> Query `editOperation.getAfter()` (Has this block been changed *during* this operation?)
> -> [IF NOT FOUND] Query `editOperation.getBefore()` (What was this block *at the start* of this operation?)
> -> [IF NOT FOUND] Query `delegate.getChunk().getBlock()` (What is the block in the *live world*?)
> -> Return Block ID to Script

