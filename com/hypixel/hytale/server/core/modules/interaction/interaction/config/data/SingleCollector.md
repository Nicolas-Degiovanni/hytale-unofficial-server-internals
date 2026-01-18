---
description: Architectural reference for SingleCollector
---

# SingleCollector<T>

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Transient State Object

## Definition
```java
// Signature
public class SingleCollector<T> implements Collector {
```

## Architecture & Concepts
The SingleCollector is a specialized, stateful implementation of the Collector interface designed for "first-match-wins" scenarios within the server's interaction system. Its primary role is to process a sequence of potential interaction targets and capture the *very first* valid result it encounters, immediately signaling completion.

This component acts as a short-circuiting mechanism. A higher-level system, such as an InteractionManager, iterates through potential candidates (e.g., entities in a raycast, blocks in an area). For each candidate, it invokes the SingleCollector's *collect* method. The collector uses a behavior-defining TriFunction, provided during its construction, to evaluate the candidate. If the function returns a non-null result, the SingleCollector stores it and returns true, instructing the calling system to halt further iteration. This pattern is highly efficient for use cases where only the first valid target is required.

The other methods defined by the Collector interface, such as *into*, *outof*, and *finished*, are intentionally left as no-ops, reinforcing this class's singular focus on finding one specific result.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a parent interaction processing system. It is not a managed service or singleton. The specific behavior of the collector is defined at creation time by injecting a TriFunction instance.
- **Scope:** The lifecycle is extremely short and is strictly bound to a single interaction evaluation operation. It is created, used once to process a set of candidates, and then becomes eligible for garbage collection.
- **Destruction:** The object is intended to be discarded after its result is retrieved via the *getResult* method. There is no explicit cleanup or destruction logic.

## Internal State & Concurrency
- **State:** The SingleCollector is fundamentally mutable. Its core purpose is to manage the internal *result* field, which transitions from null to a non-null value upon a successful collection. The *start* method must be called to reset this state before any new collection operation begins.

- **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread. It is designed for synchronous, sequential processing within a single game tick or interaction event. Concurrent calls to the *collect* method from multiple threads will result in a race condition, leading to unpredictable and corrupt final results. No internal locking mechanisms are present.

## API Surface
The public API is minimal, reflecting its role as a state machine for a single operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResult() | T | O(1) | Returns the captured result, or null if no result was collected. |
| start() | void | O(1) | Resets the internal state, setting the stored result to null. **CRITICAL:** Must be called before starting a new collection cycle. |
| collect(tag, context, interaction) | boolean | O(F) | Executes the injected function. If the function returns a non-null value, it is stored internally and this method returns true. Otherwise, returns false. The complexity O(F) depends entirely on the injected function. |

## Integration Patterns

### Standard Usage
The intended pattern involves creating an instance for a single, complete interaction query. The caller is responsible for managing the iteration and respecting the boolean signal from the *collect* method.

```java
// Example: Find the first valid entity in a list
TriFunction<CollectorTag, InteractionContext, Interaction, Entity> findFirstEntity = (tag, ctx, inter) -> {
    // Complex logic to validate if this interaction yields a valid entity
    return isValid(ctx) ? ctx.getTargetEntity() : null;
};

SingleCollector<Entity> collector = new SingleCollector<>(findFirstEntity);
collector.start(); // Prepare for collection

for (Interaction potentialInteraction : interactionList) {
    if (collector.collect(tag, context, potentialInteraction)) {
        break; // Stop iterating as soon as the first result is found
    }
}

Entity foundEntity = collector.getResult(); // Retrieve the captured entity
if (foundEntity != null) {
    // Process the result
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse without Reset:** Reusing a SingleCollector for a new operation without first calling *start* will cause it to hold stale data from the previous operation, leading to incorrect results.
- **Ignoring the `collect` Return Value:** Continuing to iterate and call *collect* after it has already returned true is inefficient and violates the class's design contract. The first result will be overwritten by subsequent successful collections.
- **Concurrent Modification:** Accessing a single instance from multiple threads is unsafe and will lead to race conditions on the internal *result* field. Each thread of execution requires its own dedicated SingleCollector instance.

## Data Pipeline
The SingleCollector is a processing stage, not a data source or sink. It acts as a stateful filter within a larger data flow.

> Flow:
> Interaction Event -> Interaction Processor iterates through candidates -> **SingleCollector.collect(candidate)** -> Internal *result* field is populated -> Processor stops iteration -> Processor retrieves value with **SingleCollector.getResult()**

