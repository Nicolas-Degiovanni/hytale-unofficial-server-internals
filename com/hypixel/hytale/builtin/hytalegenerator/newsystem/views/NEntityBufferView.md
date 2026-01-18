---
description: Architectural reference for NEntityBufferView
---

# NEntityBufferView

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.views
**Type:** Transient

## Definition
```java
// Signature
public class NEntityBufferView implements EntityContainer {
```

## Architecture & Concepts

The NEntityBufferView is a high-level facade that provides a spatial, coordinate-aware interface to a subsection of the world's entity data. It is not a data container itself; rather, it acts as a "view" or "lens" into a larger, more complex data structure known as an NBufferBundle.

Its primary architectural role is to decouple world generation algorithms from the underlying chunked storage system. A generator can request a view for a specific 3D volume (its area of interest) and perform operations like adding or iterating entities using world-space coordinates (voxel grid). The NEntityBufferView handles the internal translation of these coordinates into the appropriate buffer grid coordinates and dispatches the operations to the correct underlying NEntityBuffer instances.

This pattern allows multiple, independent generation passes to operate on the same data store without needing to manage the low-level details of buffer boundaries and lookups. It presents a contiguous, simplified 3D space to its client, abstracting away the partitioned reality of the data storage.

## Lifecycle & Ownership

-   **Creation:** An NEntityBufferView is instantiated on-demand by a world generation process. Its constructor requires a valid NBufferBundle.Access.View, which acts as a handle to the backing data store. It is never created in a vacuum; it is always tied to a pre-existing NBufferBundle.

-   **Scope:** This is a short-lived, transient object. Its lifetime is strictly bound to the scope of a specific, granular task, such as a single procedural generation pass. It is considered disposable and is not intended to be cached, serialized, or persisted.

-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources or file handles, so there is no explicit cleanup or close method. It becomes eligible for garbage collection as soon as the task that created it completes and all references to it are dropped.

## Internal State & Concurrency

-   **State:** The view's own state is **immutable** after construction. The reference to the underlying data access layer and the calculated boundary volumes are declared final and set only once in the constructor. However, the data it provides a view *into* is highly mutable. All write operations, such as addEntity, are passed through to the underlying NEntityBuffer objects, modifying their state.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms like locks or atomic operations. All interactions with an NEntityBufferView instance must be confined to the single thread that owns the world generation task.

    **WARNING:** Sharing an NEntityBufferView or its underlying NBufferBundle across multiple threads without external, explicit locking will lead to severe data corruption, race conditions, and non-deterministic behavior. The world generation system must enforce a single-threaded access model for any given region.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntity(EntityPlacementData) | void | O(1) | Calculates the target buffer from the entity's world position and delegates the add operation. Throws an assertion error if the entity is outside the view's bounds. |
| forEach(Consumer) | void | O(N) | Iterates through every entity within the view's bounds by scanning all underlying buffers. N is the total number of entities in the volume. |
| copyFrom(NEntityBufferView) | void | O(M) | Performs a bulk copy of all entities from a source view. M is the number of grid buffers within the view's volume. Asserts that the source bounds fully contain this view's bounds. |
| isInsideBuffer(x, y, z) | boolean | O(1) | Performs a fast bounds check to determine if the given voxel-grid coordinates are within this view. |

## Integration Patterns

### Standard Usage

The intended use is for a generation stage to acquire a view for its designated operational area, perform all its entity modifications, and then immediately discard the view object.

```java
// A generator acquires a view for its region of interest
NBufferBundle.Access.View access = worldGenContext.getBufferBundleAccess();
NEntityBufferView entityView = new NEntityBufferView(access);

// The generator creates and places an entity
EntityPlacementData newTree = createTreeEntity();
entityView.addEntity(newTree);

// The view is now out of scope and can be garbage collected.
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Caching:** Do not store and reuse NEntityBufferView instances across major generation stages or game ticks. They are cheap to create and represent a transactional scope. Holding onto them can prevent underlying buffers from being unloaded or managed correctly.

-   **Cross-Thread Sharing:** Never pass an NEntityBufferView instance from one thread to another. The lack of internal locking makes this pattern inherently unsafe and will corrupt the world data.

-   **Modification during Iteration:** Do not call addEntity on a view while iterating over it with forEach. The behavior is undefined and may result in a ConcurrentModificationException or missed updates.

## Data Pipeline

The NEntityBufferView serves as a key routing and access component within the world generation data pipeline. It can act as both a data sink (for writing) and a data source (for reading).

**As a Sink (Writing Entities):**
> Flow:
> Generation Algorithm -> EntityPlacementData -> **NEntityBufferView.addEntity()** -> NEntityBuffer -> NBufferBundle

**As a Source (Reading Entities):**
> Flow:
> NBufferBundle -> NEntityBuffer -> **NEntityBufferView.forEach()** -> Consumer Lambda -> Downstream Logic

