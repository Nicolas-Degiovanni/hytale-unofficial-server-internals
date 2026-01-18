---
description: Architectural reference for HashSetIntCondition
---

# HashSetIntCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class HashSetIntCondition implements IIntCondition {
```

## Architecture & Concepts
The HashSetIntCondition is a concrete implementation of the IIntCondition strategy interface. Its purpose is to provide a high-performance, set-based predicate for integer evaluation within the server's procedural world generation systems.

This class acts as a specialized filter. It encapsulates a pre-defined collection of integer identifiers—such as block types, biome IDs, or entity IDs—and determines if a given integer is a member of that collection. By leveraging the `fastutil` library's IntSet, it achieves average O(1) time complexity for membership tests, which is critical for performance-sensitive operations that are executed millions of times during world generation.

It is a foundational component for creating complex placement rules, where a generator must quickly decide if a location's properties match a set of allowed values.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by higher-level world generation services, such as feature placers or biome decorators. The caller is responsible for constructing and populating the IntSet that is injected into the constructor.
-   **Scope:** The object's lifetime is typically brief and confined to a specific, granular generation task. For example, an instance might be created to validate potential ore block placements within a single chunk, and then be discarded once that task is complete. It is not a long-lived, session-scoped object.
-   **Destruction:** Managed exclusively by the Java Garbage Collector. Once the owning generator or rule engine completes its operation and drops its reference, the HashSetIntCondition instance becomes eligible for collection.

## Internal State & Concurrency
-   **State:** The primary state is a single, final reference to an IntSet. The reference itself is immutable post-construction. However, the IntSet object it points to may be mutable. This class provides no internal mechanisms for modifying the set.
-   **Thread Safety:** This class is **conditionally thread-safe**. If the IntSet provided at construction is treated as effectively immutable (i.e., it is not modified by any thread after being passed to the constructor), then instances are safe for concurrent read-only access from multiple world generation worker threads. The class itself performs no locking.

    **Warning:** The `getSet` method exposes the internal data structure. Any external modification to the returned set after the condition has been shared across threads will result in catastrophic, non-deterministic behavior. The immutability of the backing set is a contract enforced by the caller.

## API Surface
The public contract is minimal, focusing exclusively on evaluation and state inspection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int i) | boolean | O(1) avg | The core contract method. Returns true if the integer `i` is present in the backing set. |
| getSet() | IntSet | O(1) | Returns a direct reference to the internal IntSet. Use with extreme caution. |

## Integration Patterns

### Standard Usage
This component is designed to be used as a predicate within a larger algorithm. A generator creates the condition with a specific ruleset and then repeatedly calls `eval` to test candidate values.

```java
// Example: A rule that only allows placement on stone or dirt blocks
IntSet allowedBlocks = new IntOpenHashSet(new int[]{BLOCK_STONE, BLOCK_DIRT});
IIntCondition placementCondition = new HashSetIntCondition(allowedBlocks);

// In the core generation loop...
for (int blockId : candidateBlocks) {
    if (placementCondition.eval(blockId)) {
        // ...proceed with placement logic
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Post-Construction State Mutation:** The most severe anti-pattern is modifying the backing set after the condition has been created and potentially shared. This violates its implicit contract of immutability and will lead to race conditions.

    ```java
    // DO NOT DO THIS
    IntSet sharedSet = new IntOpenHashSet();
    HashSetIntCondition condition = new HashSetIntCondition(sharedSet);
    
    // Another thread or later code modifies the set
    sharedSet.add(SOME_NEW_BLOCK_ID); // This causes non-deterministic behavior
    ```

-   **Overhead for Trivial Cases:** For conditions involving only one or two integers, the overhead of creating a HashSet may be unnecessary. A simple lambda or a dedicated class with direct equality checks (`val == A || val == B`) can be more performant and readable.

## Data Pipeline
The component acts as a decision point in a data flow, not as a data transformer. It receives an integer and outputs a boolean, which alters the control flow of its parent process.

> Flow:
> World Generation Algorithm -> Provides Candidate Integer (e.g., Block ID) -> **HashSetIntCondition.eval()** -> Returns Boolean -> Generator Branches Logic (e.g., Place Feature / Skip)

