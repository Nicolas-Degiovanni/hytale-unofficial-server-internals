---
description: Architectural reference for TreeCollector
---

# TreeCollector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Transient State Machine

## Definition
```java
// Signature
public class TreeCollector<T> implements Collector {
```

## Architecture & Concepts
The TreeCollector is a stateful builder that implements the Collector interface. Its primary role is to translate a linear sequence of traversal events into a hierarchical tree data structure. It functions as a state machine, driven by an external entity—typically a parser or walker processing Hytale's interaction configuration files.

This class decouples the act of tree traversal from tree construction. An external system walks a source data structure and emits signals like `start`, `into` (descend), `collect` (process node), and `outof` (ascend). The TreeCollector listens to these signals and builds a corresponding in-memory tree of Node objects.

The logic for generating the data payload for each node is externalized via a `TriFunction` provided during construction. This makes the TreeCollector a generic and reusable component, concerned only with structural assembly, not the semantic meaning of the data it contains.

## Lifecycle & Ownership
- **Creation:** A TreeCollector is instantiated on-demand by a system responsible for parsing a specific interaction configuration. The caller must provide a `TriFunction` that defines the transformation from a raw interaction context to the desired node data type T.

- **Scope:** The lifecycle of a TreeCollector instance is intentionally short, scoped to a single, complete collection operation. It is created, used to build exactly one tree, and then its result is retrieved via `getRoot`.

- **Destruction:** Once the root node of the generated tree has been retrieved, the TreeCollector instance is no longer needed and should be considered stale. It holds no external resources and is eligible for garbage collection as soon as it falls out of scope.

**WARNING:** A TreeCollector instance is not reusable. Attempting to call `start` a second time on the same instance will discard the previously built tree and lead to unpredictable behavior if the old tree reference is still in use. Always create a new instance for each collection task.

## Internal State & Concurrency
- **State:** The TreeCollector is fundamentally a mutable object. Its internal state, represented by the `root` and `current` node pointers, is continuously modified by the sequence of `into`, `collect`, and `outof` calls. The entire purpose of the class is to build and manage this internal tree state.

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded execution. The internal `current` cursor and the array resizing logic within the `into` method will cause race conditions, data corruption, and likely throw exceptions if a single instance is accessed concurrently from multiple threads.

**WARNING:** All calls to a TreeCollector instance must be confined to the thread that created it. Do not share instances across threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TreeCollector(function) | constructor | O(1) | Creates a new collector with the specified data-mapping function. |
| start() | void | O(1) | Resets and initializes the builder, creating the root node. Must be called first. |
| into(context, interaction) | void | O(N) | Descends one level in the tree, creating a new child node. Complexity is due to array copy for children. |
| collect(tag, context, interaction) | boolean | O(1) + F | Populates the current node's data by invoking the provided `TriFunction`. F is the complexity of the function. |
| outof() | void | O(1) | Ascends one level in the tree, moving the internal cursor to the parent node. |
| getRoot() | Node<T> | O(1) | Returns the root node of the fully constructed tree. Call this after the collection is finished. |

## Integration Patterns

### Standard Usage
The TreeCollector is intended to be driven by a walker or parser. The client code orchestrates the calls to the collector based on the structure of the source data.

```java
// Hypothetical walker driving the collector
InteractionWalker walker = new InteractionWalker();
TriFunction<CollectorTag, InteractionContext, Interaction, MyNodeData> mappingFunction = (tag, ctx, inter) -> {
    return new MyNodeData(ctx.getTargetEntity());
};

TreeCollector<MyNodeData> collector = new TreeCollector<>(mappingFunction);

// The walker invokes collector methods during its process
walker.walk(interactionConfig, collector);

// Retrieve the final result after the walk is complete
TreeCollector.Node<MyNodeData> resultTree = collector.getRoot();
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not call `start` on an existing collector to re-use it. This will orphan the previously created tree and can lead to subtle bugs if the old reference is still held elsewhere. Create a new `TreeCollector` for each distinct parsing operation.

- **Invalid Call Order:** The sequence of calls must be logical. Calling `outof` more times than `into` will result in a `NullPointerException` as the `current` pointer attempts to move above the root.

- **Concurrent Modification:** Do not access the collector from multiple threads. The internal state is not protected by locks and will become corrupted.

## Data Pipeline
The TreeCollector acts as a structural aggregator in a larger data processing pipeline.

> Flow:
> Interaction Config Source -> Configuration Walker -> **TreeCollector** (receives `into`, `collect`, `outof` events) -> In-Memory Tree (`Node<T>`) -> Server Game Logic

