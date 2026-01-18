---
description: Architectural reference for DefaultBlockMaskCondition
---

# DefaultBlockMaskCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class DefaultBlockMaskCondition implements BlockMaskCondition {
```

## Architecture & Concepts
The DefaultBlockMaskCondition is a foundational component within the server-side world generation framework. It represents the most primitive implementation of the **BlockMaskCondition** strategy interface. Its sole purpose is to provide a static, unconditional boolean result, effectively acting as a constant *true* or *false* gate within a larger chain of procedural logic.

This class is not intended for complex evaluation. Instead, it serves as a terminator, a default case, or a placeholder in world generation algorithms where a condition must be satisfied but no actual evaluation is necessary. For example, it can be used to create a rule that *always* allows a block to be placed or *never* allows it, regardless of the surrounding world state.

Two pre-allocated static instances, DEFAULT_TRUE and DEFAULT_FALSE, are provided to represent these common use cases without requiring repeated object allocation.

### Lifecycle & Ownership
- **Creation:** Instantiated in two primary ways:
    1. By higher-level world generation systems that require a constant boolean condition.
    2. By accessing the static final instances **DEFAULT_TRUE** or **DEFAULT_FALSE** for common, unconditional logic.
- **Scope:** The static instances are application-scoped and persist for the lifetime of the server. Dynamically created instances are transient and should be considered short-lived, typically existing only for the duration of a single world generation operation.
- **Destruction:** As a simple value object with no external resources, instances are managed entirely by the Java Garbage Collector.

## Internal State & Concurrency
- **State:** Immutable. The internal boolean *result* is a final field set only once during construction. The object's state cannot be modified post-creation.
- **Thread Safety:** This class is unconditionally thread-safe. Its immutability guarantees that it can be safely shared and accessed by multiple world generation worker threads without locks or other synchronization primitives.

## API Surface
The public contract is minimal, consisting of the constructor and the interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DefaultBlockMaskCondition(boolean) | constructor | O(1) | Creates a new instance with a fixed boolean result. |
| eval(int, int, BlockFluidEntry) | boolean | O(1) | Ignores all input parameters and returns the pre-configured boolean result. |

## Integration Patterns

### Standard Usage
For unconditional logic, always prefer the static instances to prevent unnecessary object creation. This is the most common and performant way to use this class.

```java
// Example: A world generation rule that always permits an action.
BlockMaskCondition condition = DefaultBlockMaskCondition.DEFAULT_TRUE;

// The result of this evaluation will always be true.
boolean canPlace = condition.eval(blockId, fluidId, nextEntry);
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Avoid creating new instances when the static singletons suffice. This pattern creates unnecessary garbage collector pressure in performance-critical world generation loops.

  ```java
  // AVOID THIS
  BlockMaskCondition condition = new DefaultBlockMaskCondition(true);

  // PREFER THIS
  BlockMaskCondition condition = DefaultBlockMaskCondition.DEFAULT_TRUE;
  ```

- **Contextual Misinterpretation:** Do not operate under the assumption that this class evaluates its inputs. It is designed to be completely ignorant of the world state passed to the *eval* method. Using it where a dynamic check is needed is a design error.

## Data Pipeline
This component does not participate in a data transformation pipeline. Instead, it acts as a simple gate in a **control flow** pipeline.

> Flow:
> World Generation Algorithm -> Evaluates Condition Chain -> **DefaultBlockMaskCondition.eval()** -> Returns Constant Boolean -> Algorithm Continues/Terminates Based on Result

