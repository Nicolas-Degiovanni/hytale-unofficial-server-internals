---
description: Architectural reference for IBlockFluidCondition
---

# IBlockFluidCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Functional Interface / Strategy Pattern

## Definition
```java
// Signature
public interface IBlockFluidCondition {
   boolean eval(int var1, int var2);
}
```

## Architecture & Concepts
The IBlockFluidCondition interface defines a fundamental contract within the procedural generation library. It serves as a predicate for evaluating the state of a specific location in the world based on its block and fluid properties. This component is a classic implementation of the **Strategy Pattern**, decoupling high-level generation algorithms from the specific conditional logic required to make decisions.

By abstracting the evaluation logic, procedural rules can be composed of various conditions without modification. For example, a cave generation algorithm can use different IBlockFluidCondition implementations to decide whether to carve through solid stone, stop at water, or avoid specific ore types. This promotes modularity and reusability within the world generation pipeline.

**WARNING:** Implementations of this interface are expected to be highly performant, as the eval method may be invoked millions of times during the generation of a single world chunk.

## Lifecycle & Ownership
As an interface, IBlockFluidCondition itself does not have a lifecycle. The following pertains to concrete implementations of this contract.

- **Creation:** Implementations are typically instantiated as stateless singletons, anonymous inner classes, or lambda expressions at the point where a procedural rule is defined or configured. They are not managed by a central registry.
- **Scope:** The lifetime of an implementation is bound to the procedural rule or generator that references it. For lambdas, the scope is often transient, existing only for the duration of a method call. For singleton implementations, the scope is application-wide.
- **Destruction:** Instances are subject to standard Java Garbage Collection. They are destroyed when no longer referenced by any part of the procedural generation system.

## Internal State & Concurrency
- **State:** The contract implicitly assumes that all implementations are **stateless**. Storing mutable state within an IBlockFluidCondition implementation is a severe anti-pattern that can lead to non-deterministic and unpredictable world generation.
- **Thread Safety:** Implementations **must be unconditionally thread-safe**. The procedural generation engine is heavily multi-threaded and will invoke the eval method from multiple worker threads concurrently without external locking. A stateless implementation is inherently thread-safe.

## API Surface
The public contract consists of a single method designed for high-frequency evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int blockId, int fluidId) | boolean | O(1) | Evaluates a condition against the provided block and fluid state identifiers. Returns true if the condition is met, false otherwise. |

## Integration Patterns

### Standard Usage
Implementations are typically provided as lambdas to procedural algorithms that require conditional logic. This allows for concise and inline definition of generation rules.

```java
// A procedural rule that only acts on solid, non-water blocks.
// The lambda is a concrete implementation of IBlockFluidCondition.
proceduralRule.setCondition((blockId, fluidId) -> {
    return Block.isSolid(blockId) && fluidId == Fluid.NONE;
});

// The rule engine later invokes the condition
if (condition.eval(currentBlock, currentFluid)) {
    // ... proceed with generation
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Storing counters, caches, or any other mutable state within an implementation will break concurrent world generation.

    ```java
    // DO NOT DO THIS
    class BadCondition implements IBlockFluidCondition {
        private int callCount = 0; // Mutable state
        public boolean eval(int blockId, int fluidId) {
            this.callCount++; // Race condition, non-deterministic
            return blockId == Block.STONE;
        }
    }
    ```
- **Heavy Computation:** The eval method must return almost instantaneously. Do not perform file I/O, network requests, or complex algorithmic calculations within an implementation. Defer such logic to higher-level systems.

## Data Pipeline
IBlockFluidCondition acts as a conditional gate within the procedural generation data flow. It does not transform data but rather directs the flow based on its boolean output.

> Flow:
> Procedural Generator queries World State -> Block & Fluid IDs -> **IBlockFluidCondition.eval()** -> Boolean Result -> Generation Algorithm makes a decision (e.g., Place Feature, Skip Coordinate)

