---
description: Architectural reference for DelimiterDouble
---

# DelimiterDouble<V>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.delimiters
**Type:** Transient

## Definition
```java
// Signature
public class DelimiterDouble<V> {
```

## Architecture & Concepts
The DelimiterDouble class is a fundamental, immutable data structure used within the procedural world generation system. It serves as a generic tuple, associating a continuous numerical range, represented by a RangeDouble, with a discrete, categorical value of type V.

Its primary role is to act as a rule or a boundary definition within a larger generation algorithm. For instance, a collection of DelimiterDouble objects can define a biome map where specific elevation ranges (the RangeDouble) map to specific biome types (the value V). By processing a noise value against a list of these delimiters, the generator can make deterministic decisions about the type of content to place at a given world coordinate.

This class is a core component for translating continuous data, such as Perlin noise output, into the discrete, state-based information required to build the game world, such as block types, vegetation densities, or climate zones.

### Lifecycle & Ownership
-   **Creation:** Instances are created directly via the public constructor: `new DelimiterDouble(...)`. This typically occurs during the initialization phase of a world generator, where a static or data-driven configuration defines the set of rules for a specific generation pass.
-   **Scope:** The lifetime of a DelimiterDouble instance is tied to its owning configuration object. It is a value object with no independent lifecycle; it exists only as part of a larger collection of rules.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are reclaimed when the generator configuration that contains them is no longer in scope. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** The DelimiterDouble class is **deeply immutable**. Both the internal *range* and *value* fields are declared final and are assigned only once at construction. This design guarantees that an instance's state cannot change after it has been created.
-   **Thread Safety:** As a direct consequence of its immutability, DelimiterDouble is **inherently thread-safe**. Instances can be safely read and shared across multiple threads without any external synchronization or locking mechanisms. This is a critical feature for high-performance, parallelized world generation tasks where multiple chunks may be processed concurrently using the same set of generation rules.

## API Surface
The public contract is minimal, consisting only of accessors for its immutable state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRange() | RangeDouble | O(1) | Returns the non-null numerical range for this delimiter. |
| getValue() | V | O(1) | Returns the nullable value associated with the range. |

## Integration Patterns

### Standard Usage
DelimiterDouble is almost always used in collections, such as a List or an array, to define a gradient or a set of strata. A common pattern is to iterate over this collection to find the specific delimiter that contains a given input value.

```java
// Example: Determining a biome from a noise value
double noiseValue = noise.get(x, z); // e.g., 0.65

List<DelimiterDouble<BiomeType>> biomeMap = buildBiomeMap();

BiomeType resultBiome = BiomeType.DEFAULT;
for (DelimiterDouble<BiomeType> delimiter : biomeMap) {
    if (delimiter.getRange().contains(noiseValue)) {
        resultBiome = delimiter.getValue();
        break;
    }
}

// The 'resultBiome' is now the biome corresponding to the noise value.
```

### Anti-Patterns (Do NOT do this)
-   **Using a Mutable Generic Type:** The most severe anti-pattern is to instantiate DelimiterDouble with a mutable type for V (e.g., `DelimiterDouble<List<String>>`). While the DelimiterDouble instance itself is immutable (its reference to the list cannot change), the internal state of the list *can* be modified. This breaks the contract of thread safety and can lead to unpredictable race conditions in a multi-threaded generator. **WARNING:** Always use immutable types for the generic value V.
-   **Mismanaging Overlapping Ranges:** The class itself does not prevent the creation of overlapping ranges in a collection of delimiters. Logic that consumes a list of DelimiterDouble objects must be prepared to handle ambiguity if ranges overlap. This can lead to non-deterministic or order-dependent generation results if not handled carefully.

## Data Pipeline
DelimiterDouble is not an active component in a pipeline but rather the configuration data that *guides* a pipeline stage. It represents a rule used by a transformation step.

> Flow:
> Generator Configuration -> List of **DelimiterDouble** rules -> Noise Function Output (double) -> Rule-Matching Logic -> Selected Value (V) -> World Builder Stage

