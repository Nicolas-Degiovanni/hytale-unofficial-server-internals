---
description: Architectural reference for ListCollector
---

# ListCollector<T>

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Transient

## Definition
```java
// Signature
public class ListCollector<T> implements Collector {
```

## Architecture & Concepts
The ListCollector is a concrete implementation of the Collector interface, designed to accumulate results from an interaction query into a standard Java List. It serves as a fundamental building block within the server's Interaction System, which processes how entities and the world interact.

Architecturally, this class embodies the **Strategy Pattern**. It is instantiated with a transformation function (a TriFunction) that defines *how* to convert a successful interaction event into an object of type T. This decouples the generic mechanism of list accumulation from the specific, domain-level logic of what data to extract from an interaction.

For example, one ListCollector instance might be configured with a function to extract entity IDs, while another might be configured to extract block coordinates from the same set of interaction events. The ListCollector itself remains agnostic to the data it is collecting.

## Lifecycle & Ownership
The lifecycle of a ListCollector is intentionally brief and tied to a single, discrete query operation. It is a stateful, single-use object.

-   **Creation:** A ListCollector is instantiated on-demand by a higher-level system responsible for executing an interaction query, such as an InteractionProcessor or a script-defined behavior. It is created immediately before the query begins.
-   **Scope:** The object's lifetime is strictly bound to the duration of the interaction query it serves. It exists only to accumulate results for that one operation.
-   **Destruction:** Once the query is complete and the results have been retrieved via the getList method, the ListCollector instance is no longer referenced by the processing system and becomes eligible for garbage collection. There is no explicit cleanup or pooling mechanism.

## Internal State & Concurrency
-   **State:** The ListCollector is highly stateful. Its primary internal state is the `list` field, which is mutated by the `collect` method on each successful interaction check. The state is reset to an empty list by a call to `start`, which is a critical first step in its lifecycle.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded execution within the scope of one interaction query. The internal list, an ObjectArrayList, is not synchronized.
    -   **WARNING:** Sharing a ListCollector instance across multiple threads will lead to race conditions, resulting in a corrupted list and unpredictable behavior. All lifecycle methods (start, collect, finished) must be called from the same thread.

## API Surface
The public contract is defined by the Collector interface, with `getList` being the primary method for retrieving results. The constructor is the main configuration point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ListCollector(function) | constructor | O(1) | Creates a new collector with a specific transformation strategy. |
| getList() | List<T> | O(1) | Returns the internal list containing all collected items. Returns null if `start` has not been called. |
| start() | void | O(1) | Prepares the collector for a new operation by creating a new internal list. **Must be called first.** |
| collect(tag, context, interaction) | boolean | O(1) + func | Executes the transformation function and adds the result to the internal list. Always returns false to signal that collection should continue. |

*Note: The methods `into`, `outof`, and `finished` are part of the Collector interface but are no-ops in this implementation.*

## Integration Patterns

### Standard Usage
The ListCollector is not intended to be used in isolation. It is passed to an interaction processing system that manages its lifecycle. The user typically only provides the transformation function and consumes the final list.

```java
// 1. Define the transformation logic
TriFunction<CollectorTag, InteractionContext, Interaction, Entity> entityExtractor = 
    (tag, context, interaction) -> context.getTargetEntity();

// 2. Create the collector with the desired logic
ListCollector<Entity> entityCollector = new ListCollector<>(entityExtractor);

// 3. The Interaction System (hypothetical) runs the query
//    This system is responsible for calling start(), collect() repeatedly, and finished().
InteractionResult result = InteractionSystem.query(someQuery, entityCollector);

// 4. Retrieve the final, populated list
List<Entity> foundEntities = entityCollector.getList();
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not hold a reference to a ListCollector and re-use it for a subsequent query without calling `start`. This will append new results to the old list, causing data corruption. Always create a new instance for each distinct query.
-   **Manual Lifecycle Management:** Avoid calling the `start`, `collect`, and `finished` methods directly unless you are implementing a custom interaction processor. Rely on the engine's interaction systems to manage the collector's lifecycle correctly.
-   **Accessing List During Collection:** Do not call `getList` and attempt to read from the list while the interaction query is still in progress. The state of the list is only guaranteed to be final after the processing system has completed its work.

## Data Pipeline
The ListCollector acts as a terminal stage in a data processing flow, converting a stream of events into a final, aggregated list.

> Flow:
> Raw Interaction Event -> Interaction Processor -> **ListCollector.collect()** -> Internal List<T> -> Consumer via getList()

