---
description: Architectural reference for IntConditionBuilder
---

# IntConditionBuilder

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Transient Builder

## Definition
```java
// Signature
public class IntConditionBuilder implements IntConsumer {
```

## Architecture & Concepts
The IntConditionBuilder is a high-performance, memory-conscious factory for creating `IIntCondition` instances. Its primary role is to aggregate a set of integer values—typically representing identifiers like block types or biome IDs—and construct an optimized condition object used in the procedural world generation system.

The core architectural principle is **lazy evaluation for performance optimization**. The builder is designed to handle three common scenarios with maximum efficiency:
1.  **Empty Set:** If no integers are added, no memory is allocated for a collection.
2.  **Single Element:** If only one unique integer is added, it avoids creating a full hash set, instead wrapping the single value in a highly optimized singleton set upon finalization.
3.  **Multiple Elements:** Only when a second unique integer is added does it allocate a full `IntSet` using a provided factory.

By implementing the `IntConsumer` interface, this class integrates seamlessly with the Java Streams API, allowing for a declarative and functional style of construction from data sources.

## Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by a higher-level system, such as a configuration parser or a feature placement algorithm, that needs to build a single condition. It is a short-lived, single-use object.
-   **Scope:** The builder's lifecycle is confined to the scope of the method in which it is created. It is intended to be used and discarded immediately after the `buildOrDefault` method is invoked.
-   **Destruction:** The object becomes eligible for garbage collection as soon as its `buildOrDefault` method has returned. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The IntConditionBuilder is a stateful object. Its internal fields, `first` and `set`, are mutated with each call to `add` or `accept`. This internal state represents the condition being constructed.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single thread. Concurrent modification from multiple threads will result in race conditions, data loss, and an incorrect final state. All operations on a single instance must be externally synchronized or confined to a single thread.

## API Surface
The public API is minimal, focusing exclusively on the construction and finalization of the condition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int value) | void | O(1) Amortized | Adds an integer to the set. This method allows the builder to be used as a terminal operation in a stream. |
| add(int value) | boolean | O(1) Amortized | Adds an integer to the set. Returns true if the internal collection was modified as a result of the call. |
| buildOrDefault(IIntCondition) | IIntCondition | O(1) | Finalizes the construction. Returns an optimized `IIntCondition` or the supplied default if no values were added. |

## Integration Patterns

### Standard Usage
The builder should be instantiated, populated, and used to build a condition within a single, contained block of logic.

```java
// Standard pattern for building a condition from a list of block IDs.
// The builder is created, used, and then discarded.

// 1. Provide a factory for the backing set and a sentinel null value.
IntConditionBuilder builder = new IntConditionBuilder(IntOpenHashSet::new, -1);

// 2. Populate the builder from a data source.
streamOfBlockIds.forEach(builder);

// 3. Finalize the build, providing a fallback condition.
IIntCondition isReplaceableBlock = builder.buildOrDefault(IIntCondition.FALSE);

// The resulting condition is now an immutable, optimized object ready for use.
if (isReplaceableBlock.test(currentBlockId)) {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
-   **Builder Reuse:** Do not invoke `add` or `accept` on an instance after `buildOrDefault` has been called. The builder does not reset its internal state, and further use will result in undefined behavior. Create a new instance for each new condition.
-   **Concurrent Access:** Do not share an instance of IntConditionBuilder across multiple threads without external locking. This will corrupt its internal state.
-   **Storing the Builder:** Do not store an IntConditionBuilder in a field or long-lived object. Its purpose is transient, and holding a reference to it after the build is complete is a memory leak.

## Data Pipeline
The IntConditionBuilder acts as a terminal aggregator in a data flow, transforming a stream of primitive integers into a single, immutable, and optimized object.

> Flow:
> Source of Integers (e.g., Config File, Stream) -> `add()` / `accept()` -> **IntConditionBuilder** (Mutable Internal State) -> `buildOrDefault()` -> Immutable `IIntCondition` Object

