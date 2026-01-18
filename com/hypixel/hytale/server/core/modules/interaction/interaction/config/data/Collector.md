---
description: Architectural reference for the Collector interface, a core contract in the server-side interaction system.
---

# Collector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface Collector {
    // Methods
}
```

## Architecture & Concepts
The Collector interface defines a stateful contract for processing and aggregating the results of an interaction query. It functions as a specialized Visitor or a state machine driven by the server's core interaction module. When a player or entity initiates an interaction, the system queries for potential targets (e.g., entities, blocks in a certain radius). An implementation of Collector is then used to traverse these potential targets, evaluate them against a given InteractionContext, and "collect" the valid results.

Its primary role is to decouple the process of *finding* potential interactions from the process of *acting* upon them. This allows for different collection strategies, such as "find the first valid interaction", "find all valid interactions", or "find the highest priority interaction", without altering the core query logic.

The lifecycle methods—start, into, collect, outof, finished—imply a traversal over a hierarchical or grouped structure of potential interaction targets.

## Lifecycle & Ownership
As an interface, Collector itself has no lifecycle. The following applies to any concrete implementation of this contract.

- **Creation:** An instance of a Collector implementation is created on-demand by the InteractionModule at the beginning of a specific interaction query. It is a short-lived, transient object.
- **Scope:** The object's lifetime is strictly bound to a single, atomic interaction query. It is created, used to process the query results, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is destroyed by the garbage collector once the interaction query method that created it returns and all references to it are dropped.

**WARNING:** Collector instances are single-use. Reusing a Collector for a subsequent interaction query will lead to state corruption and unpredictable game behavior.

## Internal State & Concurrency
- **State:** While the interface is stateless, any non-trivial implementation is expected to be highly **mutable and stateful**. Its core purpose is to accumulate state—such as a list of valid Interaction objects—as the `collect` method is invoked.
- **Thread Safety:** Implementations of this interface are **not thread-safe** and must not be assumed to be. The interaction system guarantees that a single Collector instance will only ever be used by the game thread processing the specific interaction query. Confine all access to the owning thread.

**WARNING:** Do not pass a Collector instance to other threads or asynchronous tasks. The internal state is not protected by locks and is designed for synchronous, single-threaded access.

## API Surface
The API defines a strict operational sequence for processing an interaction query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(1) | Prepares the Collector for a new collection pass. Must be called first. |
| into(context, interaction) | void | O(1) | Signals entry into a new group or context of interactions. |
| collect(tag, context, interaction) | boolean | O(1) | Processes a single potential interaction. Returns true to continue collecting, false to stop immediately. |
| outof() | void | O(1) | Signals exit from the current group or context. Balances a previous call to `into`. |
| finished() | void | O(1) | Finalizes the collection process. Called after all potential targets have been visited. |

## Integration Patterns

### Standard Usage
A developer's primary interaction with this system is to *implement* the Collector interface to define a custom filtering or aggregation strategy. The engine's InteractionModule is responsible for creating and driving the implementation.

```java
// Example of a custom implementation to find the first valid interaction
public class FindFirstCollector implements Collector {
    private Interaction result = null;

    @Override
    public void start() {
        this.result = null;
    }

    @Override
    public boolean collect(@Nonnull CollectorTag tag, @Nonnull InteractionContext context, @Nonnull Interaction interaction) {
        if (interaction.isValid()) {
            this.result = interaction;
            return false; // Stop collecting, we found one
        }
        return true; // Keep collecting
    }
    
    // Other methods implemented...

    public Optional<Interaction> getResult() {
        return Optional.ofNullable(this.result);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not manually call the lifecycle methods on a Collector. The interaction system is the sole driver of the Collector's state machine. Calling `collect` without a preceding `start` will fail.
- **State Leakage:** Do not design implementations that retain state between different interaction queries. The `start` method must fully reset the internal state of the Collector.
- **Ignoring the Return Value:** The boolean return from `collect` is a critical control signal. The interaction system uses it to perform short-circuiting optimizations. An implementation that always returns true when it could have stopped is inefficient.

## Data Pipeline
The Collector is a terminal stage in the interaction data pipeline, responsible for filtering and aggregating potential targets into a final result set.

> Flow:
> Player Input -> Server Network Handler -> InteractionModule::executeQuery -> Target Discovery -> **Collector::collect** -> Final Interaction Result -> Game Logic Execution

