---
description: Architectural reference for DelimiterInt
---

# DelimiterInt<V>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.delimiters
**Type:** Transient

## Definition
```java
// Signature
public class DelimiterInt<V> {
```

## Architecture & Concepts
The DelimiterInt class is a fundamental, immutable data structure used extensively within the world generation system. It serves as a generic container that associates a specific integer range, represented by a RangeInt object, with an arbitrary value of type V.

Its primary architectural role is to define boundaries for procedural generation rules. For example, a generator might use a collection of DelimiterInt objects to map elevation ranges to specific biome types, or temperature ranges to different block palettes. It is a core component for creating layered, rule-based environmental logic. By encapsulating a range and a value, it provides a clean, reusable primitive for defining conditional zones within a one-dimensional integer space.

## Lifecycle & Ownership
- **Creation:** Instances are created on-the-fly by world generation algorithms or deserialized from configuration files that define generation rules. They are not managed by a central registry or service container.
- **Scope:** The lifecycle of a DelimiterInt instance is typically ephemeral. It exists for the duration of a specific generation task and is then eligible for garbage collection. It is a value object, not a long-lived service.
- **Destruction:** Cleanup is managed entirely by the Java Virtual Machine's garbage collector. There are no native resources or explicit disposal methods.

## Internal State & Concurrency
- **State:** The class is **deeply immutable**. Both the internal `range` and `value` fields are marked as `final` and are assigned only once during construction. All state is established at instantiation and can never be modified.
- **Thread Safety:** DelimiterInt is unconditionally **thread-safe**. Its immutability guarantees that multiple threads can read its state concurrently without any risk of data corruption or race conditions. No synchronization mechanisms are required.

## API Surface
The public contract is minimal, consisting only of the constructor and data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRange() | RangeInt | O(1) | Returns the non-null integer range. |
| getValue() | V | O(1) | Returns the value associated with the range. May be null. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a collection of DelimiterInt objects to represent a set of rules. A generator then queries this collection to find the rule that applies to a given input value.

```java
// Example: Defining biome rules based on elevation
RangeInt plainsRange = new RangeInt(64, 80);
RangeInt mountainRange = new RangeInt(81, 128);

// BiomeType would be the generic type V
List<DelimiterInt<BiomeType>> biomeRules = List.of(
    new DelimiterInt<>(plainsRange, BiomeType.PLAINS),
    new DelimiterInt<>(mountainRange, BiomeType.MOUNTAINS)
);

// Generator logic would then find the matching rule for a given elevation
int currentElevation = 75;
BiomeType targetBiome = findBiomeForRule(biomeRules, currentElevation);
```

### Anti-Patterns (Do NOT do this)
- **Null Range:** The constructor is annotated to require a non-null RangeInt. Passing a null value will result in a runtime exception and is a violation of the class's contract.
- **State Mutation:** Do not attempt to modify the internal state of a DelimiterInt instance using reflection. The immutability guarantee is central to its design and thread safety.

## Data Pipeline
DelimiterInt is not a processing component but rather a data payload within the generation pipeline. It represents a configured rule that influences the flow of data.

> Flow:
> Generator Configuration -> **new DelimiterInt<V>(range, value)** -> Stored in Rule Collection -> Queried by Generator Logic -> Extracted V value -> Used for World Block Placement

