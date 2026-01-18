---
description: Architectural reference for HashSetBlockFluidCondition
---

# HashSetBlockFluidCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Transient

## Definition
```java
// Signature
public class HashSetBlockFluidCondition implements IBlockFluidCondition {
```

## Architecture & Concepts
The HashSetBlockFluidCondition is a high-performance predicate component used within the server-side procedural world generation engine. Its primary function is to determine if a given block and fluid pair meets a specific, pre-defined criterion.

Architecturally, this class embodies the Strategy pattern. It encapsulates a single rule—set membership—behind the generic IBlockFluidCondition interface. This allows world generation algorithms to be configured with various condition types without being coupled to their specific implementations.

The core design centers on a significant performance optimization. Instead of storing pairs of integers, it leverages MathUtil.packLong to combine a 32-bit block ID and a 32-bit fluid ID into a single 64-bit long. This packed long is then used as the key in a fastutil LongSet. This approach dramatically reduces memory overhead and garbage collector pressure compared to a set of wrapper objects, and it provides an average O(1) lookup complexity, which is critical for the performance demands of world generation.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand by higher-level world generation systems, such as a biome or structure definition loader. The caller is responsible for constructing and populating the LongSet that will be injected into the constructor. This class does not create its own state.
- **Scope:** The object is typically short-lived. Its lifetime is bound to the specific generation task that requires it, such as the population of a single world chunk or the evaluation of a specific feature placement rule.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the world generation task that created it completes and no longer holds a reference to it. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The primary state is the `set` field, a reference to a LongSet. This reference is `final` and cannot be changed after construction. The design assumes that the injected set is populated once before being passed to the constructor and is not modified thereafter. While the LongSet itself may be technically mutable, treating it as immutable post-construction is critical for system stability.

- **Thread Safety:** This class is thread-safe under the strict condition that the underlying LongSet is not modified after construction. The `eval` method is a pure read-only operation. Given that world generation is a heavily multi-threaded process, multiple worker threads can safely and concurrently call `eval` on a shared instance of HashSetBlockFluidCondition without locks or synchronization, provided the immutability contract is honored.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int block, int fluid) | boolean | O(1) | Evaluates if the block/fluid pair exists in the internal set. This is the primary operational method. |
| getSet() | LongSet | O(1) | Returns a direct reference to the internal set. **Warning:** Modifying the returned set is a severe anti-pattern. |

## Integration Patterns

### Standard Usage
This component is intended to be instantiated by a configuration or rule engine and passed to an algorithm that evaluates it. The creator of the condition is responsible for pre-calculating the set of valid block/fluid pairs.

```java
// A world feature rule needs to check for specific ground conditions.
// The set is prepared once during configuration loading.
LongSet validGroundTypes = new LongOpenHashSet();
validGroundTypes.add(MathUtil.packLong(BLOCK_ID_GRASS, FLUID_ID_NONE));
validGroundTypes.add(MathUtil.packLong(BLOCK_ID_DIRT, FLUID_ID_NONE));

// The condition is created and can be reused for many evaluations.
IBlockFluidCondition groundCondition = new HashSetBlockFluidCondition(validGroundTypes);

// A generator uses the condition to make a decision.
if (groundCondition.eval(world.getBlock(x, y, z), world.getFluid(x, y, z))) {
    // Place the feature
}
```

### Anti-Patterns (Do NOT do this)
- **Post-Construction State Mutation:** The most critical anti-pattern is modifying the underlying LongSet after the condition has been created and shared. Accessing the set via `getSet()` and calling `add` or `remove` can lead to race conditions and non-deterministic world generation, as different threads may observe different states of the condition.

- **Empty Set Instantiation:** Creating an instance with an empty set is functionally valid (it will always evaluate to false), but it is inefficient. A dedicated `FalseCondition` implementation should be used instead to more clearly signal intent and avoid unnecessary object allocation.

## Data Pipeline
This class acts as a conditional gate or predicate within a larger data flow. It does not transform data; it produces a boolean result that directs the flow.

> Flow:
> World Generation Engine -> Provides (block ID, fluid ID) -> **HashSetBlockFluidCondition.eval** -> Returns boolean -> World Generation Engine -> (Places Feature | Skips Feature)

