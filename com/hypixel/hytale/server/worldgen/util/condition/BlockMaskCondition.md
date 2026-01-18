---
description: Architectural reference for BlockMaskCondition
---

# BlockMaskCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Functional Interface (Strategy Pattern)

## Definition
```java
// Signature
@FunctionalInterface
public interface BlockMaskCondition {
   boolean eval(int var1, int var2, BlockFluidEntry var3);
}
```

## Architecture & Concepts
The BlockMaskCondition interface is a fundamental component within the server-side procedural world generation framework. It formalizes a contract for conditional evaluation against a specific block in the world, acting as a predicate in generation algorithms.

This interface embodies the **Strategy Pattern**, decoupling high-level generation logic (e.g., ore placement, terrain shaping) from the specific rules that determine *where* that logic applies. Instead of hard-coding checks like *if block is stone*, generators are configured with implementations of BlockMaskCondition. This allows for the creation of highly flexible, reusable, and composable generation rules that can be combined to produce complex environmental features.

Its primary role is to answer a single question for a given coordinate: "Should the generation algorithm operate on this block?"

### Lifecycle & Ownership
- **Creation:** Implementations are typically instantiated as lambda expressions or method references at the point of use within a higher-level world generation service, such as a ZoneGenerator or FeaturePlacer. They are not managed by a central registry or dependency injection container.
- **Scope:** The lifetime of a BlockMaskCondition instance is extremely ephemeral. It is created as a transient object, passed as an argument to a generation method, and becomes eligible for garbage collection as soon as that method completes its execution for a given chunk or region.
- **Destruction:** Managed entirely by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Implementations of this interface are **required** to be stateless. The `eval` method's result must depend solely on its input arguments. Capturing and relying on mutable external state within a lambda implementation is a severe anti-pattern that will lead to unpredictable generation results.
- **Thread Safety:** The interface itself is inherently thread-safe. However, thread safety is the responsibility of the implementation. Given that world generation is a massively parallel process, all implementations **must be re-entrant and stateless** to guarantee correct and deterministic behavior when executed by multiple worker threads simultaneously.

## API Surface
The contract consists of a single method, designed for high-frequency invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int x, int z, BlockFluidEntry block) | boolean | O(1) | Evaluates the condition for a block at the given local coordinates. Returns true if the condition is met, false otherwise. |

**WARNING:** The integer arguments represent local coordinates within the current processing slice (e.g., a chunk), not global world coordinates. The BlockFluidEntry provides access to the block and fluid state at that position.

## Integration Patterns

### Standard Usage
The interface is intended to be implemented using a lambda expression for concise, inline rule definition. This implementation is then passed to a generation utility that iterates over a set of blocks.

```java
// A generator method receives a condition to filter which blocks to modify.
public void replaceStoneWithOre(Chunk chunk, BlockMaskCondition condition) {
    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            BlockFluidEntry entry = chunk.getBlock(x, z);
            // The condition is evaluated for every block.
            if (condition.eval(x, z, entry)) {
                chunk.setBlock(x, z, ORE_BLOCK);
            }
        }
    }
}

// Usage: Pass a lambda that checks if the block is stone.
BlockMaskCondition isStone = (x, z, block) -> block.is(STONE);
generator.replaceStoneWithOre(currentChunk, isStone);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that rely on or modify external state. This will break determinism and cause severe race conditions in the multi-threaded generator.
  ```java
  // BAD: Modifies external state, not thread-safe.
  int blocksFound = 0;
  BlockMaskCondition statefulCondition = (x, z, block) -> {
      if (block.is(STONE)) {
          blocksFound++; // Race condition!
          return true;
      }
      return false;
  };
  ```
- **Expensive Computations:** The `eval` method is in the hot path of world generation and will be called millions of times. Avoid any logic with high computational complexity, disk I/O, or network requests. Conditions must be lightweight and execute instantly.

## Data Pipeline
BlockMaskCondition acts as a control-flow gate within the world generation data pipeline, not as a data transformer.

> Flow:
> World Generator -> Iterates Block Coordinates (x, z) -> Fetches BlockFluidEntry -> **BlockMaskCondition.eval()** -> [true/false] -> Conditional Block Transformation

