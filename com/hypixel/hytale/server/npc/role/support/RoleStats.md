---
description: Architectural reference for RoleStats
---

# RoleStats

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class RoleStats {
```

## Architecture & Concepts
RoleStats is a transient, stateful data container designed to aggregate spatial or statistical data during a single server tick for an NPC's role-based AI calculation. It acts as a temporary workspace, collecting numerical data categorized by entity type (Player or NPC) and by a specific semantic context defined by the RangeType enum.

The primary purpose of this class is to serve as an intermediate data structure. A higher-level AI system, such as a RoleProcessor, populates a RoleStats instance with relevant data from the current game state. This data, typically representing distances or other metrics, is then consumed by decision-making algorithms to determine NPC behavior, such as positioning, target selection, or threat assessment.

Its design, using primitive-specialized collections from fastutil, indicates a focus on performance and memory efficiency, which is critical for server-side AI systems that may run calculations for hundreds or thousands of NPCs simultaneously.

## Lifecycle & Ownership
- **Creation:** An instance of RoleStats is created on-demand by a parent system responsible for executing an NPC's role logic. It is a plain Java object, not managed by a dependency injection framework or service locator.

- **Scope:** The lifetime of a RoleStats object is extremely short, typically confined to a single AI processing cycle or even a single function call. It is created, populated, read, and then immediately becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. The presence of a `clear` method suggests that instances may be pooled and reused in performance-critical scenarios to reduce object allocation overhead, though this is an implementation detail of the owning system.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. The core function of the class is to accumulate data via the `trackRange` and `trackBuckets` methods. The state is composed of several maps and lists that cache this data for the duration of a calculation.

- **Thread Safety:** This class is **not thread-safe**. All internal collections are non-synchronized. Concurrent modification from multiple threads will result in data corruption or runtime exceptions. It is designed to be confined to a single thread, almost certainly the main server thread responsible for ticking a specific NPC's AI.

**WARNING:** Do not share instances of RoleStats across threads. All population and read operations must be performed by the same thread.

## API Surface
The public API is designed for a simple populate-then-read workflow.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clear() | void | O(N) | Resets all internal collections, preparing the instance for reuse. N is the number of range types tracked. |
| trackRange(isPlayer, type, value) | void | Amortized O(1) | Adds a single integer value to the appropriate set based on entity and range type. |
| getRanges(isPlayer, type) | IntSet | O(1) | Returns the raw, unordered set of tracked ranges. The returned set must not be modified. |
| getRangesSorted(isPlayer, type) | int[] | O(N log N) | Returns a new, sorted array of tracked ranges. This is a computationally expensive operation. |
| trackBuckets(isPlayer, bucketRanges) | void | O(1) | Overwrites the current bucket list with the provided list. |
| getBuckets(isPlayer) | IntArrayList | O(1) | Returns the stored list of bucket ranges. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate, populate, and consume the object within a single, short-lived scope.

```java
// An AI system creates a new instance for a specific calculation
RoleStats stats = new RoleStats();

// Populate with data from the game world
for (Player player : nearbyPlayers) {
    int distance = calculateDistance(npc, player);
    stats.trackRange(true, RoleStats.RangeType.UNSORTED, distance);
}

// Consume the aggregated data to make a decision
int[] sortedPlayerRanges = stats.getRangesSorted(true, RoleStats.RangeType.UNSORTED);
if (sortedPlayerRanges != null && sortedPlayerRanges[0] < DANGER_THRESHOLD) {
    npc.performEvasiveAction();
}

// The 'stats' object is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use Without Clearing:** Reusing a RoleStats instance across multiple AI ticks or for different NPCs without calling `clear()` will lead to data contamination and incorrect AI behavior.
- **Repetitive Sorting:** Calling `getRangesSorted` multiple times within the same algorithm is inefficient. If the sorted data is needed more than once, cache the result in a local variable.
- **Concurrent Access:** Accessing a single RoleStats instance from multiple threads is strictly forbidden and will lead to an unstable server state.

## Data Pipeline
RoleStats is a central component in a micro-pipeline for AI decision-making. It transforms unstructured world data into a structured, queryable format for algorithmic consumption.

> Flow:
> Raw Game State (Entity Positions) -> AI Role Processor -> **RoleStats (Aggregation)** -> AI Decision Logic -> NPC Action Command

